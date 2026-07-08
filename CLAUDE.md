# GN IMS Options Flow Dashboard

Personal live-market dashboard for Nifty 50 options trading (India, NSE).

## Architecture

- **This repo**: a single self-contained standalone HTML document, [index.html](index.html). Vanilla JS, no build step, no framework, no dependencies to install. It is a pure read-only client — it never writes to Supabase, only fetches. Deployed as-is to GitHub Pages (or any static host) with no build process.
- **Charting**: TradingView's own [Lightweight Charts](https://www.tradingview.com/lightweight-charts/) library (loaded via CDN, global `LightweightCharts`), used for all 5 panels — native candlesticks, scroll-to-zoom, drag-to-pan, pinch-zoom. Charts are created once and persist across live refreshes (only `series.setData()` is called on poll), so pan/zoom position survives.
- **Styling**: standalone dark theme inspired by TradingView (hardcoded CSS custom properties in the `<style>` block, Inter font via Google Fonts CDN) — not dependent on any host page's design tokens.
- **Data source**: Supabase Postgres, read via the REST API (`/rest/v1/...`) using the anon/publishable key hardcoded in the script. This is intentional, not a leak — Row Level Security is enabled on all tables with SELECT-only policies, so the key can only ever read. Safe to expose in a public repo.
- **Data pipeline (outside this repo)**: an n8n workflow running on the user's own VPS pulls Nifty 50 options chain data from the Upstox API every 3 minutes during NSE market hours (9:15–15:30 IST) and computes derived analytics (max pain, PCR, OI walls, OI velocity, GEX, rolling ATM premium, etc.), writing the results into Supabase. That pipeline is not part of this project and should not be built or modified here.
- **Tables consumed**:
  - `options_snapshots` — one row per 3-minute cycle, aggregate metrics (spot price, PCR, call/put wall strikes, call/put OI velocity, ATM strike/straddle price, and optionally `atm_ce_ltp`/`atm_pe_ltp` — the code detects a 400 from Postgrest if those columns don't exist yet and falls back to a reduced select).
  - `nifty_ohlc_3min` — OHLC candles, same 3-minute cadence.

## Important notes on current behavior — do not treat these as bugs

- The main price chart falls back to a plain spot-price line only when there's no OHLC data yet for the selected range (`ohlcRows.length === 0`). There is no "broken render" detection anymore — that was a workaround for a bug in the old `chartjs-chart-financial` library, which has been replaced.
- The "Setup Signal" panel is a placeholder (dashes, "Placeholder" badge). It is not wired to real detection logic. When that logic is eventually built, it belongs server-side in the n8n workflow — not in this dashboard.
- Live mode polls every 45 seconds and auto-tracks the latest `snapshot_date` present in Supabase; a date-range picker can switch to a fixed historical range instead.
- Crosshair and time-scale (zoom/pan) are synced across all 5 panels using Lightweight Charts' native `subscribeCrosshairMove`/`setCrosshairPosition` and `subscribeVisibleLogicalRangeChange`/`setVisibleLogicalRange` APIs. Touch pan/zoom is native to the library — there's no custom long-press handler anymore.
- ATM strike-roll markers use Lightweight Charts' native series marker API (`setMarkers`), not a custom drawing plugin.

## Deployment

- Hosted for free on GitHub Pages, serving directly from the repo root (`index.html`) — no build step, push to the tracked branch to update the live site.
- Local dev/testing: `python -m http.server 8532` from the project root, then open `http://localhost:8532/index.html` (or the machine's LAN IP instead of `localhost` to test from a phone on the same Wi-Fi).

## Working conventions

- No build step — edit `index.html` directly.
- Do not refactor or restructure without being asked; the current file is a validated working baseline.
- Feature work (additional analytics panels, the server-side setup-detection system) will be directed incrementally — this file will be updated as that context accumulates.
