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

## Toggle / preset system

- **Two separate toggle rows**: "Overlays" (Group 1 → lines/band on the main chart) and "Panels" (Groups 2–4 + the existing velocity/ATM panels → whole panes below). Rows are generated from `overlayToggleDefs` / `panelToggleDefs`; a unified handler flips `toggles[key]`, applies the single effect, clears the active-preset highlight, and persists.
- **Presets** (`presetDefs`): Structure, Momentum, Volatility, Everything. A preset sets its listed keys ON and every other toggle OFF (Everything = all on) via `applyPreset`. Presets are a convenience layer only — manual toggles keep working and deselect the preset chip.
- **Persistence**: the whole `toggles` object is saved to `localStorage` under `gnims_toggles_v1` (`saveToggles`/`loadToggles`), wrapped in try/catch so it degrades to in-memory state where storage is unavailable (e.g. an artifact sandbox). On the hosted GitHub Pages site, localStorage works normally.
- **Default view is deliberately strict-minimal**: candles + Call Wall + Put Wall only. Everything else — including the pre-existing OI-velocity and ATM-premium panels — starts OFF on a clean first load (no saved state). This "which preset should be the default primary view" decision is intentionally deferred; do **not** auto-commit a preset as the default.

## Important notes on current behavior — do not treat these as bugs

- The main price chart falls back to a plain spot-price line only when there's no OHLC data yet for the selected range (`ohlcRows.length === 0`). There is no "broken render" detection anymore — that was a workaround for a bug in the old `chartjs-chart-financial` library, which has been replaced. **Keep this fallback.**
- The "Setup Signal" panel is a placeholder (dashes, "Placeholder" badge). It is not wired to real detection logic. When that logic is eventually built, it belongs server-side in the n8n workflow — not in this dashboard.
- Live mode polls every 45 seconds and auto-tracks the latest `snapshot_date` present in Supabase; a date-range picker can switch to a fixed historical range instead.
- Crosshair and time-scale (zoom/pan) are synced across every pane using Lightweight Charts' native `subscribeCrosshairMove`/`setCrosshairPosition` and `subscribeVisibleLogicalRangeChange`/`setVisibleLogicalRange` APIs. Touch pan/zoom is native to the library — there's no custom long-press handler anymore. Mobile tap-to-track crosshair is enabled via `trackingMode.exitMode: OnNextTap`.
- ATM strike-roll markers use Lightweight Charts' native series marker API (`setMarkers`), not a custom drawing plugin.
- The snapshot fetch uses `select=*` on purpose (see Tables consumed). Do **not** switch it back to an explicit column list — that reintroduces the HTTP-400-on-missing-column fragility when the pipeline adds new analytics columns.
- Optional panels are created lazily on first toggle-on and kept in `syncGroup` thereafter; hidden panes are just `display:none`. When a pane is shown, its width is re-applied and its visible range synced to the main chart (`resizeEntry` + `applyCurrentRange`), because a `display:none` container reports width 0 at build time.

## Deployment

- Hosted for free on GitHub Pages, serving directly from the repo root (`index.html`) — no build step, push to the tracked branch to update the live site.
- Local dev/testing: `python -m http.server 8532` from the project root, then open `http://localhost:8532/index.html` (or the machine's LAN IP instead of `localhost` to test from a phone on the same Wi-Fi).

## Working conventions

- No build step — edit `index.html` directly.
- Do not refactor or restructure without being asked; the current file is a validated working baseline.
- **Adding a new overlay or panel is data-driven**: append an entry to `overlayLineDefs` + `overlayToggleDefs` (main-chart overlay) or to `panelsConfig` + `panelToggleDefs` (new pane). No new bespoke builder functions needed — `ensurePanel`/`renderPanel` and the shared sync wiring handle it.
- Feature work (additional analytics panels, the server-side setup-detection system) will be directed incrementally — this file will be updated as that context accumulates.

## Invariants a future session must NOT break

- Read-only. The dashboard never writes to Supabase; keep it that way.
- Keep the candlestick → spot-line fallback.
- Keep the snapshot fetch on `select=*` (incremental-column resilience).
- GEX (and any future huge-magnitude metric) is scaled **in the display layer only** — never mutate the stored value.
- Setup Signal stays a placeholder; detection logic lives server-side in n8n, not here.
- The default first-load view stays strict-minimal (candles + Call/Put walls). The choice of a default "primary" preset is deliberately deferred — don't auto-apply one.
