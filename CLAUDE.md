# GN IMS Options Flow Dashboard

Personal live-market dashboard for Nifty 50 options trading (India, NSE).

## Architecture

- **This repo**: a single self-contained HTML fragment, [gn_ims_options_flow_dashboard_v3_1.html](gn_ims_options_flow_dashboard_v3_1.html). Vanilla JS, no build step, no framework. It is a pure read-only client — it never writes to Supabase, only fetches.
- **Data source**: Supabase Postgres, read via the REST API (`/rest/v1/...`) using the anon/publishable key hardcoded in the script. This is intentional, not a leak — Row Level Security is enabled on all tables with SELECT-only policies, so the key can only ever read.
- **Data pipeline (outside this repo)**: an n8n workflow running on the user's own VPS pulls Nifty 50 options chain data from the Upstox API every 3 minutes during NSE market hours (9:15–15:30 IST) and computes derived analytics (max pain, PCR, OI walls, OI velocity, GEX, rolling ATM premium, etc.), writing the results into Supabase. That pipeline is not part of this project and should not be built or modified here.
- **Tables consumed**:
  - `options_snapshots` — one row per 3-minute cycle, aggregate metrics (spot price, PCR, call/put wall strikes, call/put OI velocity, ATM strike/straddle price, and optionally `atm_ce_ltp`/`atm_pe_ltp` — the code detects a 400 from Postgrest if those columns don't exist yet and falls back to a reduced select).
  - `nifty_ohlc_3min` — OHLC candles, same 3-minute cadence.

## Important notes on current behavior — do not treat these as bugs

- The file is an HTML **fragment** (no `<!DOCTYPE>`, `<html>`, `<head>`, or `<body>`), meant to be embedded in a host page that supplies the `--bg-accent` / `--text-*` / `--surface-*` design-token CSS variables and the `ti ti-*` (Tabler) icon font. Opening it standalone in a browser will look unstyled — that's expected until it's embedded in its host, not something to fix.
- The main price chart tries candlesticks via `chartjs-chart-financial` first, and automatically falls back to a plain spot-price line if the library fails to load or the y-axis renders obviously broken (`yMax < 1000`). This fallback is intentional and should stay.
- The "Setup Signal" panel is a placeholder (dashes, "Placeholder" badge). It is not wired to real detection logic. When that logic is eventually built, it belongs server-side in the n8n workflow — not in this dashboard.
- Live mode polls every 45 seconds and auto-tracks the latest `snapshot_date` present in Supabase; a date-range picker can switch to a fixed historical range instead.
- Crosshair is synced across all 5 charts (`mainChart`, `atmChart`, `callVelChart`, `putVelChart`, `netVelChart`): mouse hover on desktop, 2-second long-press-and-drag on mobile.

## Working conventions

- No build step — edit the HTML file directly.
- Do not refactor or restructure without being asked; the current file is a validated working baseline.
- Feature work (additional analytics panels, the server-side setup-detection system) will be directed incrementally — this file will be updated as that context accumulates.
