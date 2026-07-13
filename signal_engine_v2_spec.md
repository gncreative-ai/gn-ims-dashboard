# Signal engine v2 — build spec for Claude Code

## Read first

Read the current dashboard file AND `CLAUDE.md` fully before writing any code. This spec adds a real, working signal engine on top of the existing dashboard — it does NOT touch the n8n pipeline or Supabase schema. Everything in this phase is client-side JavaScript in the dashboard, reading data that already exists.

**Scope boundary — important:** this phase computes everything in the browser, on data already in `nifty_ohlc_3min` and `options_snapshots`. Do NOT create new Supabase tables, do NOT touch n8n. That's a deliberate, separate future phase (porting this same logic server-side for Telegram alerts) — not part of this build.

---

## Background — why this exists

Two things already proven and validated (do not redesign these, implement as specified):

1. **ZLEMA is a real trigger, but noisy alone.** A raw ZLEMA cross on 3-min spot candles fires far too often (roughly every 15 minutes on a typical session) — mostly whipsaw during range chop. It needs (a) a hold-confirmation before counting as a real trigger, and (b) the options gate below to filter which triggers are worth trusting.
2. **The options chain is the confirmation gate, not the trigger.** It answers "should I trust this trigger," not "when do I enter." Keep these two jobs separate in the code — a trigger module and a gate module, never merged into one function.

---

## Part 1 — Indicator engine (swappable, multi-select)

### 1a. ZLEMA (existing, make duration editable)

```js
function indicator_ZLEMA(closes, length) {
  var lag = Math.floor((length - 1) / 2);
  var adjusted = closes.map(function(v, i){ return i >= lag ? v + (v - closes[i-lag]) : v; });
  var k = 2/(length+1);
  var out = []; var ema = adjusted[0];
  for (var i=0;i<adjusted.length;i++){ ema = i===0 ? adjusted[0] : adjusted[i]*k + ema*(1-k); out.push(ema); }
  return out;
}
```

**New requirement:** add a number input (default `24`, matching current behavior) next to the chart that lets the user change ZLEMA's `length` parameter. Changing it must recompute the trigger and redraw immediately — no page reload.

### 1b. VPC — volume-free adaptation (new)

The real VPC indicator (Pine script, provided separately) anchors its channel to volume-weighted highs/lows. **The Nifty index has no volume**, so a literal port is not valid. Build a volume-free adaptation instead:

```js
function vpc_channel(highs, lows, length){
  var upper=[], lower=[];
  for (var i=0;i<highs.length;i++){
    var s = Math.max(0, i-length+1);
    var hh=-Infinity, ll=Infinity;
    for (var j=s;j<=i;j++){ if(highs[j]>hh)hh=highs[j]; if(lows[j]<ll)ll=lows[j]; }
    upper.push(hh); lower.push(ll);
  }
  return {upper:upper, lower:lower};
}
```

**VPC trigger events (mirrors the Pine `dir` state without volume):** a BULL trigger fires when the current bar's `high` equals the rolling `upper` (a new N-period high just formed). A BEAR trigger fires when the current bar's `low` equals the rolling `lower` (a new N-period low just formed).

**Add the same length control for VPC** (default `20`, matching the Pine default) — the user only explicitly asked for ZLEMA's duration to be editable, but leaving VPC hardcoded while ZLEMA is adjustable would be an inconsistent half-measure. Add it; note this was your addition if asked.

### 1c. Indicator selection checkboxes

Place two checkboxes directly above the main chart: **"ZLEMA"** and **"VPC"**, both checked by default.

**Combination logic when both are selected — read carefully, this is a deliberate design choice, not arbitrary:**
- **ZLEMA only checked** → triggers come from ZLEMA crosses only.
- **VPC only checked** → triggers come from VPC new-high/new-low events only.
- **Both checked** → a trigger only fires when BOTH agree on direction within the same or adjacent bar (confluence/AND logic, not OR). Reasoning: we already proved a single indicator alone is noisy. Firing on *either* one would double the noise, not reduce it. Requiring agreement is consistent with every confirmation rule already built into this system (the gate, the 2-reading OI rule, etc.) — the whole project's philosophy is "demand agreement across independent signals before trusting a trigger."
- **Neither checked** → show a clear "select at least one indicator" state; do not run detection.

---

## Part 2 — Trigger confirmation (existing, keep as-is)

A raw cross/breakout is provisional until it holds for `CONFIRM_BARS` additional bars in the same direction (default `1`). Only confirmed triggers proceed to the gate. This logic is already validated — reuse it:

```js
function detectTriggers(closes, triggerSeries, CONFIRM_BARS){
  var events = [];
  for (var i=1;i<closes.length;i++){
    var prevAbove = closes[i-1] > triggerSeries[i-1];
    var nowAbove = closes[i] > triggerSeries[i];
    if (prevAbove !== nowAbove){
      var held = true;
      for (var c=1;c<=CONFIRM_BARS;c++){
        if (i+c >= closes.length){ held = false; break; }
        var stillAbove = closes[i+c] > triggerSeries[i+c];
        if (stillAbove !== nowAbove){ held = false; break; }
      }
      events.push({ idx:i, dir: nowAbove?'BULL':'BEAR', confirmed: held });
    }
  }
  return events;
}
```

---

## Part 3 — Options gate (existing, validated — reuse exactly)

For each confirmed trigger, find the nearest `options_snapshots` row at or before the trigger's timestamp, and score it:

| # | Check | BULL passes when | BEAR passes when |
|---|---|---|---|
| 1 | Regime (`spot_dist_from_max_pain`) | context only, always shown, never blocks a pass | same |
| 2 | Vs max pain | `spot_price < max_pain_strike + 100` (room to rise) | `spot_price > max_pain_strike - 100` (room to fall) |
| 3 | Wall clearance | `call_wall_strike - spot_price > 80` | `spot_price - put_wall_strike > 80` |
| 4 | GEX | `gex_total < 0` | `gex_total < 0` |
| 5 | PCR bias | `pcr_oi > 1.0` | `pcr_oi < 1.0` |

**Pass threshold: 3 of the 4 scored checks** (regime is contextual/neutral, doesn't count toward the score). These thresholds are reasoned defaults from three days of manual analysis, not yet outcome-tuned — keep them as named constants at the top of the file (`GATE_WALL_CLEARANCE = 80`, `GATE_MAXPAIN_ROOM = 100`, `GATE_PASS_THRESHOLD = 3`) so they're easy to adjust later without hunting through logic.

---

## Part 4 — Signal card UI (must match the existing Setup Signal panel exactly)

The dashboard already has a placeholder "Setup signal" panel with this exact layout: title + badge top row, then five label/value rows — **Setup, Direction, Entry, Stop loss, Target** — then a muted note line below.

**Build ONE reusable card component matching that exact layout**, and use it in two places:

1. **The live panel** (replaces today's placeholder when a signal is active): shows the most recent confirmed, gate-passed signal. Falls back to the existing placeholder text ("No active setup...") when nothing has passed.
2. **A history list below it**: every confirmed trigger today (passed or not), most recent first, using the same card layout, small and stacked.

**Field mapping — how to fill Setup/Direction/Entry/Stop loss/Target from the trigger + gate data:**

- **Setup:** describe which indicator(s) fired, e.g. `"ZLEMA cross"`, `"VPC breakout"`, or `"ZLEMA + VPC confluence"` depending on the active selection.
- **Direction:** `Long` for BULL, `Short` for BEAR (plainer than the internal BULL/BEAR labels).
- **Entry:** the trigger bar's close price.
- **Stop loss:** for Long, the VPC channel's `lower` value at the trigger bar. For Short, the channel's `upper` value. (Structural level, not a position-sizing recommendation — see disclaimer below.)
- **Target:** the nearer of `call_wall_strike` or `max_pain_strike` (Long) / `put_wall_strike` or `max_pain_strike` (Short), whichever is closer to Entry in the trade's direction.
- **Badge (top right):** gate score, e.g. `"4/4"` or `"3/4"`, colored to match pass/fail (reuse existing success/danger tokens already in the dashboard).
- **Note line:** one sentence — which gate checks failed if not a full pass, or "Range regime — expect smaller moves" / "Trend regime — may run further" based on the regime check, if it passed.

**Important disclaimer to include in the panel itself, small muted text:** *"Stop loss and target are structural levels (channel bounds, walls, max pain) — not risk-managed position sizing. Confirm position size independently."* This matters — don't drop it.

---

## Part 5 — Collapse/expand

Add a collapse toggle (chevron icon, click to hide/show body) on the Setup Signal panel header — both the live panel and, ideally, the history list can collapse independently. Persist collapsed/expanded state in `localStorage` (this is the real hosted dashboard, not an artifact — `localStorage` is fine here, unlike in a sandboxed chat artifact).

---

## Constraints — do not violate

- Everything computes client-side in this phase. No new Supabase tables. No n8n changes.
- Keep the trigger module (Part 1) and gate module (Part 3) as separate, independently callable functions — never merge their logic. This is what makes swapping indicators later (point made explicitly by the user: "today ZLEMA/VPC, tomorrow something else") a one-function change instead of a rewrite.
- Reuse the exact gate math and confirmation logic given above — it's already validated against real data; don't redesign it.
- Don't touch the existing OI velocity, PCR, straddle, or other panels already on the dashboard.
- Match existing visual style/theme (dark surfaces, existing color tokens) — this is an addition, not a redesign.

---

## When done — update CLAUDE.md

Add a section documenting:
- The trigger/gate/card three-layer architecture and why they're separated (swappability).
- Both indicator formulas (ZLEMA and the volume-free VPC adaptation), and why VPC isn't a literal Pine port (no index volume).
- The confluence (AND) logic when both indicators are selected, and why OR was rejected.
- The full gate table and current threshold constants, flagged as "reasoned defaults, not yet outcome-tuned."
- Explicit note: **this phase is dashboard-only; a future phase ports the same trigger+gate logic into n8n with a new Supabase table for Telegram alerting — not yet started.**
