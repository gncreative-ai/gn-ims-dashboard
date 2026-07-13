# GN IMS Options Flow Dashboard

Personal live-market dashboard for Nifty 50 options trading (India, NSE).

## Architecture

- **This repo**: a single self-contained standalone HTML document, [index.html](index.html). Vanilla JS, no build step, no framework, no dependencies to install. It is a pure read-only client — it never writes to Supabase, only fetches. Deployed as-is to GitHub Pages (or any static host) with no build process.
- **Charting**: TradingView's own [Lightweight Charts](https://www.tradingview.com/lightweight-charts/) library (loaded via CDN, global `LightweightCharts`), used for every pane — native candlesticks, scroll-to-zoom, drag-to-pan, pinch-zoom. Charts are created once and persist across live refreshes (only `series.setData()` is called on poll), so pan/zoom position survives. New optional panels are created lazily the first time they're toggled on; all live panes register into a single `syncGroup` for shared crosshair + time-scale sync.
- **Styling**: standalone dark theme inspired by TradingView (hardcoded CSS custom properties in the `<style>` block, Inter font via Google Fonts CDN) — not dependent on any host page's design tokens.
- **Data source**: Supabase Postgres, read via the REST API (`/rest/v1/...`) using the anon/publishable key hardcoded in the script. This is intentional, not a leak — Row Level Security is enabled on all tables with SELECT-only policies, so the key can only ever read. Safe to expose in a public repo.
- **Data pipeline (outside this repo)**: an n8n workflow running on the user's own VPS pulls Nifty 50 options chain data from the Upstox API every 3 minutes during NSE market hours (9:15–15:30 IST) and computes derived analytics (max pain, PCR, OI walls, OI velocity, GEX, rolling ATM premium, etc.), writing the results into Supabase. That pipeline is not part of this project and should not be built or modified here.
- **Tables consumed**:
  - `options_snapshots` — one row per 3-minute cycle, all aggregate/derived metrics (see the metric catalog below). **Fetched with `select=*`** so that columns the n8n pipeline adds incrementally are picked up automatically. This is deliberate: PostgREST returns HTTP 400 for the *entire* query if you name a column that doesn't exist yet, so we never enumerate columns in the select. Every metric is null-guarded, and any metric whose column is absent (or has no data in range) simply shows a per-panel "not available yet" note.
  - `nifty_ohlc_3min` — OHLC candles, same 3-minute cadence.

## Metric catalog (what's displayed and where)

Everything below `options_snapshots` unless noted. Metrics are grouped by measurement unit — items can only share a Y-axis if they share units.

**Baseline (always-available core, not behind the new toggle groups):**
- Main chart candles ← `nifty_ohlc_3min` (open/high/low/close); falls back to a spot-price line (`spot_price`) when no OHLC rows exist for the range.
- Call Wall (`call_wall_strike`) and Put Wall (`put_wall_strike`) — dedicated overlay lines on the main chart, their own toggles (not part of `overlayLineDefs`).
- Rolling ATM Call/Put premium panel (`atm_ce_ltp`, `atm_pe_ltp`) with strike-roll markers (`atm_strike`).
- Call OI velocity (`call_oi_velocity`), Put OI velocity (`put_oi_velocity`), Net OI velocity (`call_oi_velocity − put_oi_velocity`, computed client-side as `netVel`) — histogram panels.
- Metric cards: Spot (`spot_price`), PCR OI (`pcr_oi`), ATM straddle (`atm_straddle_price`).

**GROUP 1 — Overlays on the main candle chart (share price Y-scale).** Config: `overlayLineDefs` (lines) + the band. Toggle labels in `overlayToggleDefs`.
- 2nd Call Wall `call_wall_2_strike`; 2nd Put Wall `put_wall_2_strike`; Max Pain `max_pain_strike`; Gamma Flip `gamma_flip_strike`; OI-weighted avg strike `oi_weighted_avg_strike`; Battleground `battleground_strike`; Futures price `futures_price`.
- **Expected-move band** — shaded fill between `spot_price + atm_straddle_price` (upper) and `spot_price − atm_straddle_price` (lower). Rendered by a **custom Lightweight Charts series primitive** (`createBandPrimitive`, `zOrder:'bottom'`), because the library has no native fill-between-two-lines. Attached to the main series and re-attached whenever the main chart is rebuilt (candles↔line switch).

**GROUPS 2–4 — separate synced panels below the main chart.** Config: `panelsConfig` (one entry per panel; each lists its series + optional `refLines`/`fixedRange`/`unit`/`transform`). Toggle labels in `panelToggleDefs`.
- Group 2 (ratios/oscillators, each with a horizontal reference line where noted): PCR OI `pcr_oi` + PCR Volume `pcr_volume` (same panel, ref 1.0); IV Skew `iv_skew` (ref 0); Risk Reversal `risk_reversal` (ref 0); Net Delta `net_delta` (ref 0); Net Gamma `net_gamma` (ref 0); OI Concentration `oi_concentration_ratio` (fixed 0–1 scale); Spot distance from Max Pain `spot_dist_from_max_pain` (ref 0); Aggregate Theta `aggregate_theta`; Aggregate Vega `aggregate_vega`.
- Group 3 (premium / raw counts): ATM Straddle `atm_straddle_price` (₹); ATM IV `atm_iv_call` + `atm_iv_put` (same panel, %); Basis `basis` (futures−spot, ₹, ref 0); Total Call OI `total_call_oi` + Total Put OI `total_put_oi` (same panel, contracts).
- Group 4 (GEX, isolated): GEX total `gex_total`. **Display-only scaling: divided by 1e9** via a `transform` in the panel config (unit label "×10⁹ (billions)"); the stored value is never altered.

## Panel UX (info headers, hover readouts, collapse, notes)

- **Info headers**: every panel (including the main chart) shows a `.panelInfo` line under its legend explaining what the metric is, what the line colors mean, and how to read it. For the config-driven panels this is the `info` field in `panelsConfig`; the main/ATM/velocity panels have their `info` text inline in the HTML. These are general market-structure descriptions, not trading advice (a `.disclaimer` line at the bottom of the page states this).
- **Hover value readouts** — on **every pane except the main chart**. Each panel header has a `.panelReadout` (`readout_<key>`) that shows the timestamp + each series' value at the hovered point. Driven by `updateReadouts(timeSec)`, called from the shared crosshair handler in `createSyncedChart`, so hovering any pane updates all readouts in sync. `nearestRowIndex` (binary search over `rowTimes`) maps the crosshair time to the nearest snapshot row. When not hovering, readouts show the latest row. Per-series decimals come from the `dec` field; `transform` (e.g. GEX ÷1e9) is applied before formatting. The registry is built once in `buildReadouts` (4 static panels + every `panelsConfig` entry).
- **Collapse / expand** — on **every pane except the main chart**. Each collapsible panel has `data-collapse-key` and a `.collapseBtn` (−/+) wired in `initCollapse`; collapsing adds `.collapsed` (hides `.panelBody` and the readout) and persists to `localStorage` under `gnims_collapsed_v1`. On expand, the pane is resized and its range re-synced (a `display:none` body reports width 0 while collapsed).
- **Notes**: a free-text `<textarea id="notesArea">` near the bottom, auto-saved (400 ms debounce) to `localStorage` under `gnims_notes_v1` via `initNotes`, with a "Saved" indicator. Degrades to in-memory if storage is unavailable.

## Signal engine v2 (client-side trigger + gate + card)

A real, working setup-detection engine layered on top of the existing dashboard — computed entirely in the browser from data already in `nifty_ohlc_3min` and `options_snapshots`. **This phase is dashboard-only.** A future phase ports the same trigger+gate logic into the n8n pipeline with a new Supabase table for Telegram alerting — not yet started, not part of this repo.

**Three-layer architecture, kept deliberately separate** (never merge these — this is what makes swapping/adding indicators later a one-function change instead of a rewrite):
1. **Trigger module** — `indicator_ZLEMA`, `vpc_channel`, `detectTriggers`, `detectVpcTriggers`. Decides *when* a raw signal fires.
2. **Gate module** — `scoreEvent`. Decides *whether to trust* a confirmed trigger, by scoring it against the options chain. Never decides entry timing.
3. **Card UI** — `signalCardHtml`. One reusable component rendering the Setup/Direction/Entry/Stop loss/Target layout, used for both the live panel and every history row.

**Indicators (Part 1):**
- **ZLEMA** (`indicator_ZLEMA(closes, length)`): zero-lag EMA — a lag-adjusted price series fed through a standard EMA formula. Length is user-editable (number input, default `24`), recomputes immediately on change, no reload.
- **VPC — volume-free adaptation** (`vpc_channel(highs, lows, length)`): the real VPC indicator anchors its channel to volume-weighted highs/lows; **the Nifty index has no volume**, so this is a rolling N-period high/low channel instead (`upper`/`lower`), not a literal Pine port. Length is user-editable (default `20`, matching the Pine default — added for consistency with ZLEMA's editable length even though only ZLEMA's was explicitly requested).
- **VPC trigger events** (`detectVpcTriggers`): a BULL trigger fires when the bar's `high` equals the rolling `upper` (new N-period high); BEAR when `low` equals the rolling `lower`. This differs from ZLEMA's cross-based trigger (`detectTriggers`, cross of `closes` vs the indicator series) because VPC's trigger is a breakout event, not a line cross.
- **Indicator selection**: two checkboxes above the main chart (`chkZlema`, `chkVpc`), both checked by default. **Combination logic is deliberate, not arbitrary:**
  - ZLEMA only → triggers from ZLEMA crosses only.
  - VPC only → triggers from VPC new-high/new-low events only.
  - **Both checked → confluence (AND), not OR.** A trigger only fires when both agree on direction within the same or adjacent bar (`Math.abs(idx diff) <= 1`). OR was explicitly rejected: firing on *either* indicator would double the noise a single indicator already has, not reduce it — consistent with every other confirmation rule in this system (the gate, the 2-reading OI rule, etc.).
  - Neither checked → "select at least one indicator" state, no detection runs.
  - The VPC channel (`upper`/`lower`) is always computed regardless of which indicator(s) are selected as triggers, because stop-loss levels are read from it even in "ZLEMA only" mode.

**Confirmation (Part 2, unchanged from spec):** a raw cross/breakout is provisional until it holds for `SIGNAL_CONFIRM_BARS` additional bars in the same direction (default `1`). Only confirmed triggers reach the gate.

**Options gate (Part 3, `scoreEvent`)** — for each confirmed trigger, finds the nearest `options_snapshots` row at or before the trigger's timestamp (`nearestRowAtOrBefore`, binary search over `rowTimes`) and scores it:

| # | Check | BULL passes when | BEAR passes when |
|---|---|---|---|
| 1 | Regime (`spot_dist_from_max_pain`) | context only, always shown, never blocks a pass | same |
| 2 | Vs max pain | `spot_price < max_pain_strike + GATE_MAXPAIN_ROOM` | `spot_price > max_pain_strike - GATE_MAXPAIN_ROOM` |
| 3 | Wall clearance | `call_wall_strike - spot_price > GATE_WALL_CLEARANCE` | `spot_price - put_wall_strike > GATE_WALL_CLEARANCE` |
| 4 | GEX | `gex_total < 0` | `gex_total < 0` |
| 5 | PCR bias | `pcr_oi > 1.0` | `pcr_oi < 1.0` |

Pass threshold: `GATE_PASS_THRESHOLD` (3) of the 4 scored checks (regime is contextual, doesn't count toward the score). Constants — `GATE_WALL_CLEARANCE = 80`, `GATE_MAXPAIN_ROOM = 100`, `GATE_PASS_THRESHOLD = 3` — are **reasoned defaults from manual analysis, not yet outcome-tuned**. `GATE_REGIME_RANGE_THRESHOLD = 50` (points of `spot_dist_from_max_pain`) is this session's own addition (not in the original spec) to decide the regime note's wording — also a reasoned default, easy to retune.

**Signal card (Part 4)** — `signalCardHtml(sig, small)` fills: Setup (which indicator(s) fired, per the active selection — `"ZLEMA cross"` / `"VPC breakout"` / `"ZLEMA + VPC confluence"`), Direction (`Long`/`Short`), Entry (trigger bar's close), Stop loss (VPC channel `lower` for Long / `upper` for Short, at the trigger bar — a structural level, not position sizing), Target (nearer of the relevant wall or max pain, ahead of Entry in the trade's direction), and a badge showing the gate score (e.g. `3/4`) colored pass/fail (`badgeOk`/`badgeFail` classes reusing the existing `--up`/`--down`/`--danger-bg` tokens). The note line states which checks failed on a non-pass, or the regime read (`"Range regime — expect smaller moves"` / `"Trend regime — may run further"`) on a pass. The panel carries a fixed small-print disclaimer: stop loss/target are structural levels, not risk-managed position sizing.
- **Live panel** (`#panelSetupLive` → `#setupLiveCard`): the most recent confirmed trigger that passed the gate; falls back to a "no active setup" card when none has passed, or an explicit message when no indicator is selected or no OHLC data exists yet for the range.
- **History list** (`#panelSetupHistory` → `#setupHistoryList`): every confirmed trigger in the currently loaded range, passed or not, most recent first, using the same card at a smaller size (`setupPanelSmall`).
- Both recompute via `computeSignals()`, called after every data load and on every indicator-control change (checkbox or length input) — no page reload needed.

**Collapse (Part 5):** both `panelSetupLive` and `panelSetupHistory` are plain `.panel[data-collapse-key]` elements, so they collapse/expand and persist via the *existing* `initCollapse()`/`gnims_collapsed_v1` machinery with no special-casing — `collapseEntryFor` simply returns `null` for their keys since they have no chart to resize.

**On-chart markers:** every confirmed trigger (in `scored`, regardless of gate pass/fail) is also drawn directly on the main candlestick chart via `applySignalMarkers`, using Lightweight Charts' native marker API (`mainSeries.setMarkers()` — the same mechanism as the existing ATM strike-roll markers, not a custom drawing layer). Green `arrowUp` below the bar = confirmed BULL trigger; red `arrowDown` above the bar = confirmed BEAR trigger; the marker's text is the gate score (e.g. `"3/4"`) so pass/fail is visible at a glance without opening the history list. Recomputed every time `computeSignals()` runs (data load, indicator checkbox/length change) and cleared when no indicator is selected or no OHLC data exists yet.

**On-chart ZLEMA line + VPC channel:** both indicators are also drawn directly on the main chart as their own persistent line series (`zlemaSeries`, `vpcUpperSeries`, `vpcLowerSeries`, created in `ensureMainChart` alongside the other main-chart series, rebuilt whenever the chart itself rebuilds on the candle↔spot-line fallback switch). ZLEMA renders as an amber line (`colZlema`); the VPC channel renders as two grey dashed lines (reusing `colMuted`, the dashboard's existing "reference/context" color). `updateIndicatorOverlays(zlemaVals, vpc, times)`, called from `computeSignals()`, both sets the data and toggles each series' `visible` to match its own checkbox — so unchecking VPC hides its channel on the chart exactly as it stops contributing triggers, without needing separate toggle UI. Values are computed unconditionally regardless of checkbox state (same reasoning as the stop-loss levels), only the on-chart *visibility* follows the checkbox.

## Toggle / preset system

- **Two separate toggle rows**: "Overlays" (Group 1 → lines/band on the main chart) and "Panels" (Groups 2–4 + the existing velocity/ATM panels → whole panes below). Rows are generated from `overlayToggleDefs` / `panelToggleDefs`; a unified handler flips `toggles[key]`, applies the single effect, clears the active-preset highlight, and persists.
- **Presets** (`presetDefs`): a preset sets its listed keys ON and every other toggle OFF via `applyPreset`. Presets are a convenience layer only — manual toggles keep working and deselect the preset chip. Exact membership:
  - **Structure**: Call Wall, Put Wall, 2nd Call Wall, 2nd Put Wall, Max Pain, Battleground, Expected-move band (panels off).
  - **Momentum**: Call/Put/Net OI velocity, Net Delta, PCR panel.
  - **Volatility**: ATM Straddle, ATM IV, IV Skew, GEX.
  - **Everything**: all overlays + all panels on.
- **Persistence**: the whole `toggles` object is saved to `localStorage` under `gnims_toggles_v1` (`saveToggles`/`loadToggles`), wrapped in try/catch so it degrades to in-memory state where storage is unavailable (e.g. an artifact sandbox). On the hosted GitHub Pages site, localStorage works normally.
- **Default view is deliberately strict-minimal**: candles + Call Wall + Put Wall only. Everything else — including the pre-existing OI-velocity and ATM-premium panels — starts OFF on a clean first load (no saved state). This "which preset should be the default primary view" decision is intentionally deferred; do **not** auto-commit a preset as the default.

## Important notes on current behavior — do not treat these as bugs

- The main price chart falls back to a plain spot-price line only when there's no OHLC data yet for the selected range (`ohlcRows.length === 0`). There is no "broken render" detection anymore — that was a workaround for a bug in the old `chartjs-chart-financial` library, which has been replaced. **Keep this fallback.**
- The "Setup Signal" panel is now a real, working client-side detection engine (see Signal engine v2 above) — not a placeholder. It still does not write anywhere; a future, separate phase ports the same logic server-side into n8n for Telegram alerting.
- Live mode polls every 45 seconds and auto-tracks the latest `snapshot_date` present in Supabase; a date-range picker can switch to a fixed historical range instead.
- Crosshair and time-scale (zoom/pan) are synced across every pane using Lightweight Charts' native `subscribeCrosshairMove`/`setCrosshairPosition` and `subscribeVisibleLogicalRangeChange`/`setVisibleLogicalRange` APIs. Touch pan/zoom is native to the library — there's no custom long-press handler anymore. Mobile tap-to-track crosshair is enabled via `trackingMode.exitMode: OnNextTap`.
- ATM strike-roll markers use Lightweight Charts' native series marker API (`setMarkers`), not a custom drawing plugin.
- The snapshot fetch uses `select=*` on purpose (see Tables consumed). Do **not** switch it back to an explicit column list — that reintroduces the HTTP-400-on-missing-column fragility when the pipeline adds new analytics columns.
- Optional panels are created lazily on first toggle-on and kept in `syncGroup` thereafter; hidden panes are just `display:none`. When a pane is shown, its width is re-applied and its visible range synced to the main chart (`resizeEntry` + `applyCurrentRange`), because a `display:none` container reports width 0 at build time.

## Deployment

- **Repo**: https://github.com/gncreative-ai/gn-ims-dashboard (public), default branch `main`.
- **Live site**: https://gncreative-ai.github.io/gn-ims-dashboard/ — GitHub Pages served from the repo root (`index.html`) on `main`. No build step: `git push` to `main` updates the live site within ~30–60s.
- **Public-repo security decision (deliberate)**: the repo is public and the Supabase anon key is exposed in `index.html` on purpose — RLS makes it read-only (see Data source). A login gate (Cloudflare Access, which would require hosting on Cloudflare Pages rather than GitHub Pages since Access can't front a `*.github.io` URL without a custom domain) was discussed and **intentionally deferred** — not forgotten. Revisit if the data ever becomes something worth gating.
- **Local dev/testing**: `python -m http.server 8532` from the project root, then open `http://localhost:8532/index.html` (or the machine's LAN IP instead of `localhost` to test from a phone on the same Wi-Fi). A `.claude/launch.json` defines this server for the Claude Code preview tools.

## Working conventions

- No build step — edit `index.html` directly.
- Do not refactor or restructure without being asked; the current file is a validated working baseline.
- **Adding a new overlay or panel is data-driven**: append an entry to `overlayLineDefs` + `overlayToggleDefs` (main-chart overlay) or to `panelsConfig` + `panelToggleDefs` (new pane). No new bespoke builder functions needed — `ensurePanel`/`renderPanel` and the shared sync wiring handle it. For a new panel, also set `info` (the header description) and `dec` per series (readout decimals); the info header, hover readout, and collapse control are generated automatically by `buildPanelsDOM`/`buildReadouts`/`initCollapse`.
- **The main price chart is deliberately excluded** from the hover readout and the collapse control (per design); it keeps its info header only. Don't add those two to it without being asked.
- Feature work (additional analytics panels, the server-side setup-detection system) will be directed incrementally — this file will be updated as that context accumulates.

## Invariants a future session must NOT break

- Read-only. The dashboard never writes to Supabase; keep it that way.
- Keep the candlestick → spot-line fallback.
- Keep the snapshot fetch on `select=*` (incremental-column resilience).
- GEX (and any future huge-magnitude metric) is scaled **in the display layer only** — never mutate the stored value.
- Setup Signal detection runs client-side here (see Signal engine v2); the future n8n/Telegram port is a separate phase, not a replacement for this one.
- The default first-load view stays strict-minimal (candles + Call/Put walls). The choice of a default "primary" preset is deliberately deferred — don't auto-apply one.
- Signal engine: keep the trigger module (`indicator_ZLEMA`/`vpc_channel`/`detectTriggers`/`detectVpcTriggers`) and the gate module (`scoreEvent`) as separate, independently callable functions — never merge them. The gate scores trust, it never decides entry timing.
- Signal engine stays dashboard-only (client-side, no writes). Porting the same trigger+gate logic into n8n with a new Supabase table for Telegram alerting is a distinct, not-yet-started future phase — do not build it here.
