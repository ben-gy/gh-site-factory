# Build Log: Flight Delays (AU)
**Date:** 2026-06-26
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty, so I researched uncovered Australian open-data domains. The registry already covers childcare, charities, pollution, NDIS, aged care, crime, planning, road crashes, donations, hospitals, energy, dams and fire — but **no aviation / flight data**. The BITRE "Domestic Airlines — On Time Performance" dataset on data.gov.au (modified 2026-06-23, data through May 2026) is a genuinely new domain: highly searchable, dashboard-oriented, and rich enough for a route map and network graph.

## Site Details
- **Name:** Flight Delays (AU)
- **Repo:** ben-gy/au-flights
- **Category:** transport
- **Audience:** Australian travellers comparing airlines/routes before booking; nervous flyers; frequent flyers; aviation watchers/journalists
- **Stack:** Vanilla TypeScript + Vite + Vitest
- **Data strategy:** pipeline (Node download + aggregate → JSON in public/data/, refreshed by GitHub Actions cron)

## Data Sources
- BITRE Domestic Airline On Time Performance — https://data.gov.au/data/dataset/domestic-airline-on-time-performance (CSV resource `otp_time_series_web.csv`, 9.6 MB, 118,102 rows, Jan 2004 – May 2026)
- Curated airport coordinates (44 reporting airports) embedded in `src/data/airports.ts` for the Leaflet map

## Architecture Decisions
- **Vanilla TS over React:** single-page tabbed tool; no need for component trees or routing libraries. Smaller bundle (206 KB / 62 KB gzip).
- **Pipeline over runtime-fetch:** the source is a 9.6 MB CSV with 118k rows — far too large to parse in the browser. `pipeline/collect.mjs` downloads it and `pipeline/aggregate.mjs` pre-computes compact JSON (summary, airlines, routes, airports, national series; ~0.9 MB total). A `data-pipeline.yml` Action refreshes it twice weekly.
- **Hand-rolled SVG charts:** bar charts, multi-series line charts, sparklines, the force-directed network graph, and the heatmaps are all hand-rolled — no D3/Chart.js. Leaflet is the only viz dependency (map + GeoJSON-style markers).
- **9 views** to exceed the "5 distinct views" bar: Overview, Airlines (leaderboard + per-airline history), Routes (table + slide-in drill-down), Trend (22-year), Seasonality (month-of-year + year×month heatmap), Map, Network graph, Matrix (airline × airport heatmap), Insights (auto-anomaly).
- **Airline brand colours** defined once in the pipeline and flowed through every view for consistency.

## Test Results
- Tests written: 51 (format helpers, analysis/insights, generated-data integrity)
- Tests passed: 51
- Tests failed: 0
- One initial failure was a faulty test assertion (`82.05.toFixed(1)` floating-point edge → expected 82.1 but JS yields 82.0); fixed the test value, not the code.

## Build Status
- npm install: pass
- npm test: pass (51/51)
- npm run build: pass (tsc + vite, 0 errors after fixing unused imports, a LinePoint null-narrowing type guard, and adding vite/client types)
- Local preview: pass — verified all 9 views, route drill-down, mobile layout, and no console errors via Claude Preview; fixed network edge colours (CSS `stroke` was overriding the SVG attribute → switched to inline style)

## Deployment
- Repo created: yes (ben-gy/au-flights, public)
- GitHub Pages enabled: yes (workflow build)
- Cloudflare DNS: yes (CNAME au-flights → ben-gy.github.io)
- CNAME set + cycled for cert: yes
- Deploy workflow: completed success
- Live check: https://au-flights.benrichardson.dev returns 200 over HTTPS; /data/summary.json returns 200
- TLS: serving over HTTPS at the edge; `https_enforced` flag still propagating at log time (typically flips within minutes)
- PR created: https://github.com/ben-gy/au-flights/pull/1

## Errors & Resolutions
- `aggregate.mjs` initially crashed with "Maximum call stack size exceeded" from `Math.max(...rows.map())` over 118k rows → replaced with a reduce loop.
- TypeScript build errors (unused imports, `LinePoint.y` null narrowing, `import.meta.env` types) → fixed imports, added a type guard, added `src/vite-env.d.ts`.
- Network graph edges rendered grey because the CSS `.net-edge { stroke }` rule overrode the SVG `stroke` attribute → set colour via inline `style.stroke` (wins over CSS).
