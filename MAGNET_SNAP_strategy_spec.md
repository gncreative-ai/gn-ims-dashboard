# MAGNET SNAP — strategy spec for IMS dashboard

## STATUS: HYPOTHESIS, NOT A VALIDATED EDGE. Build in LOG mode. Do not trade real money on it yet.

This is the output of a systematic search over 6 clean trading days (Jul 8–15, 2026), ~40 parameter variants across 5 strategy families. It is the **best surviving candidate**, not a proven money-maker. Read the "Evidence & limitations" section before building.

---

## 1. What it is, in one line

When spot is stretched away from Max Pain and starts to turn back, ride the snap back toward Max Pain. Max Pain is the target. ATR is the stop. Stop trading for the day after 3 losers.

This is essentially the **Magnet Pull** setup from GANEE's IMS framework, mechanised and quantified.

---

## 2. Exact rules

### Data inputs (per 3-min bar)
- `nifty_ohlc_3min`: open, high, low, close
- `options_snapshots`: `max_pain_strike`, `spot_dist_from_max_pain`

### Derived
- **ATR(14)** on 3-min bars, standard True Range:
  `TR = max(high-low, |high - prev_close|, |low - prev_close|)`, then 14-bar simple average.

### Entry — LONG
All must be true on the just-closed bar:
1. `spot_dist_from_max_pain < -25` (spot is at least 25 pts BELOW max pain)
2. `close > previous bar close` (the turn-up trigger — do not enter on a falling bar)
3. No position currently open
4. Fewer than 3 losing trades so far today
5. Time is before 15:25 IST

### Entry — SHORT
Mirror image:
1. `spot_dist_from_max_pain > +25` (spot at least 25 pts ABOVE max pain)
2. `close < previous bar close` (turn-down trigger)
3. + same conditions 3, 4, 5

### On entry — draw BOTH lines immediately
- **Entry** = close of the signal bar
- **LONG:** `SL = entry - 2.0 * ATR` , `TG = max_pain_strike`
- **SHORT:** `SL = entry + 2.0 * ATR` , `TG = max_pain_strike`

Note: TG is the **max pain strike itself** — a structural target, not an ATR multiple. This is deliberate: it's the level the trade thesis says price is being pulled toward.

### Exit — whichever comes first
1. **TG hit** — LONG: `high >= TG`. SHORT: `low <= TG`.
2. **SL hit** — LONG: `low <= SL`. SHORT: `high >= SL`.
3. **Time stop** — 40 bars (120 min) elapsed → exit at close.
4. **EOD** — 15:25 IST → exit at close, flat by 15:30.

When one line is hit, **remove both lines**. Position closed.

### Position & session rules
- **One position at a time.** No new signal is evaluated while a position is open.
- After a position closes, the very next bar can produce a new signal.
- **Daily loss limit: after 3 losing trades, stop taking new entries for the rest of the day.** (This is the single most important risk control — see evidence below.)
- Every day starts fresh: no positions carried, loss counter resets, ATR/EMA recomputed from that day's bars only.
- **Hard flat by 15:25**, no exceptions.

---

## 3. UI requirements (match existing dashboard patterns)

- Green triangle marker on the chart at LONG entries, red triangle at SHORT entries — same style as the existing signal markers.
- On an open position, draw two horizontal lines from entry bar forward: **SL line (red)** and **TG line (green)**. When either is hit, stop drawing both.
- Signal card must use the **existing Setup Signal panel layout** (Setup / Direction / Entry / Stop loss / Target rows + badge + note line).
  - **Setup:** "Magnet Snap"
  - **Direction:** Long / Short
  - **Entry / Stop loss / Target:** the three prices
  - **Note line:** e.g. "Spot 38 pts below max pain 24150, turn-up confirmed. Target = max pain."
- History list below: every trade today with entry time, direction, exit reason (TG / SL / Time / EOD), and points.
- **Add a running day counter:** trades taken, wins, losses, and "loss limit: 2/3 used". When 3 losses hit, show a clear "Daily loss limit reached — no new entries" state.
- This indicator goes in the same checkbox group as ZLEMA/VPC. When "Magnet Snap" is checked, ZLEMA/VPC triggers are ignored (it is a standalone strategy, not a confluence input).

---

## 4. Constants (put at top of file, named, easy to tune)

```js
const MS_MIN_DIST      = 25;    // min |spot - max pain| in points to arm the setup
const MS_SL_ATR        = 2.0;   // stop loss = this * ATR(14)
const MS_ATR_LEN       = 14;
const MS_MAX_HOLD_BARS = 40;    // 120 min time stop
const MS_MAX_LOSSES    = 3;     // daily loss limit
const MS_CUTOFF        = "15:25";
```

---

## 5. Evidence & limitations — READ THIS

### What the backtest actually showed (6 days, Jul 8–15 2026, 25 trades)

| Cost assumption | PF | Net (6 days) | Win% | Profitable days |
|---|---|---|---|---|
| 8 pts round-trip (conservative) | 0.97 | −10 | 59% | 4/6 |
| 5 pts round-trip (optimistic) | **1.19** | **+56** | 59% | 4/6 |

Per-day at cost=5, with the 3-loss limit:
`Jul08: −92 | Jul09: −3 | Jul10: +48 | Jul13: +34 | Jul14: +26 | Jul15: +41`

### The honest problems

1. **It is entirely dependent on the cost assumption.** At 8 points round-trip it is break-even. At 5 it is marginally positive. The real number depends on GANEE's actual fills. **This must be measured, not assumed.**
2. **It works on 5 of 6 days and gets destroyed on trend days.** July 8 (the −379 pt trend day) alone loses −92 even *with* the loss limit. Without the limit it loses −179. This is a fade strategy: it will always bleed when a real trend runs. A trend-day filter cannot be built or validated from a sample containing exactly ONE trend day — attempting it would be pure overfitting.
3. **25 trades is not a sample.** It is an anecdote. No win rate computed from 25 trades is trustworthy.
4. **The 3-loss daily limit is doing a lot of work.** It is legitimate risk management (reactive, not predictive), but its exact value (3) is fitted to 6 days and should be treated as arbitrary.

### What was tested and rejected (so it isn't re-tried blindly)

| Family | Best result | Verdict |
|---|---|---|
| Dip-buy below EMA21 | Gross PF **1.51** (+7.8 pts/trade) → net PF 0.99 | **Real gross edge, 100% consumed by costs** |
| Opening Range Breakout (15/30/45min) | PF 0.70 | Rejected |
| EMA 9/21 momentum cross | PF 0.33 | Rejected — worst of all |
| Straddle-expansion breakout | PF 0.78 | Rejected |
| Options gates (GEX/PCR/basis) added to dip-buy | PF 1.12 max on 8 trades | Rejected — no gate rescued it |

**The single most important finding:** the dip-buy has a genuine, day-consistent gross edge (6/6 days positive in the feature scan, PF 1.51 gross) that is *exactly* the size of the transaction cost. Edges of this magnitude exist in this data. They are not big enough to trade after friction.

### Baseline reality (why targets must stay modest)
- Median 3-min bar True Range: **13.8 points** — that is the noise floor.
- A *random* entry gets a median max-favourable-excursion of only **~25-30 points over 60 min**, ~33-45 over 120 min.
- Any strategy promising large scalps on this instrument at this timeframe is fantasy.

---

## 6. How to use this (the actual recommendation)

**Run it in LOG mode first.** Show the signals, draw the lines, record every outcome — but treat it as data collection, not as a trade recommendation. Specifically:

- Log every signal with: timestamp, direction, entry, SL, TG, exit reason, points gained/lost, and the options context at entry (mpdist, PCR, GEX, straddle).
- **Log the misses as carefully as the hits.** The dataset that eventually validates or kills this strategy is the one that includes every trade it *would* have taken.
- After ~30-40 sessions, there will be enough trades to compute a win rate that means something.
- **Critically: measure real fill costs.** If actual round-trip cost is closer to 8 points than 5, this strategy does not work and should be abandoned.

---

## 7. CLAUDE.md update

Document:
- The Magnet Snap rules and all constants.
- That it is a **standalone strategy** (not a ZLEMA/VPC confluence input) and mutually exclusive with them.
- That it is **an unvalidated hypothesis in LOG mode**, PF 0.97–1.19 depending on cost assumption, from a 25-trade / 6-day sample.
- The rejected-families table above, so future sessions don't blindly re-test ORB / EMA cross / straddle breakout.
- The baseline noise stats (13.8 pt median 3-min TR; ~25-30 pt random 60-min MFE) as the sanity check on any future strategy proposal.
