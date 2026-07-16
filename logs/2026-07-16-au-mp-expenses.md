# Build Log: Parliamentary Expenses
**Date:** 2026-07-16
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty, so I surveyed uncovered Australian open datasets against the registry (which is already dense in AU gov/data explorers). Shortlisted two strong, genuinely-new candidates — IPEA parliamentary expenses and NGER corporate greenhouse emissions — and chose **IPEA parliamentary expenses**: more distinctive (no expenses/parliamentarian site exists in the fleet), highly newsworthy and searchable, with a natural federal-electorate map, party colours and rich multi-view potential. NGER was rejected as the stronger *build* because it's a smaller dataset and closer in shape to the existing au-pollution (NPI) site.

## Site Details
- **Name:** Parliamentary Expenses
- **Repo:** ben-gy/au-mp-expenses
- **Category:** government-transparency
- **Audience:** journalists, political researchers, civically-engaged citizens (desktop-first, but a fast single-name lookup on mobile)
- **Stack:** Vanilla TypeScript + Vite + Leaflet + Vitest
- **Data strategy:** pipeline. Source (IPEA) publishes **quarterly**, so the cron is quarterly (`23 6 5 2,5,8,11 *`) — a few weeks after each quarter closes, staggered day/minute. Electorate boundaries are a committed static asset.

## Data Sources
- IPEA "Current and Former Parliamentarians' Expenditure" quarterly extracts, discovered via the data.gov.au CKAN API (`package_search`), main `*_dataextract.csv` per quarter. 15 quarters in window (Jul 2022 – Mar 2026).
- Digital Atlas of Australia — Federal Electoral Divisions (March 2025) FeatureServer (ArcGIS Online, `aus_digitalatlas`), 150 divisions, mapshaper-simplified to ~1.4 MB.

## Architecture Decisions
- **Vanilla TS** (not React): single page with view tabs, one Leaflet map, hand-rolled SVG charts — no need for a component framework.
- **Pipeline + committed data:** the frontend reads pre-aggregated JSON from `public/data/`. Ran collect + aggregate locally to populate for the first deploy; the quarterly workflow refreshes it. Raw transaction volume (~35k rows × 15 quarters) is aggregated server-side to per-person / per-category / per-quarter / per-electorate JSON so the browser payload stays small (~365 KB parliamentarians.json + 18 KB summary + per-MP chunks).
- **Dependency-free CSV parser** in the pipeline (quoted fields, embedded newlines, doubled quotes) so the data-pipeline workflow needs no `npm ci`.
- **Data window:** from the 2022 federal election (2022 Q3) onward — an editorial scope, clearly labelled, that keeps current members prominent while still spanning the 2025 election.
- **Boundary sourcing:** ABS ASGS ArcGIS `query` endpoint rejected all queries (HTTP 400); pivoted to the Digital Atlas March 2025 FeatureServer, which matched all 150 current divisions (only abolished Higgins & North Sydney unmapped — expected).
- **Data correctness check:** verified that "Aggregated Total" rows (Employee Travel, Office Facilities) do not double-count itemised rows — each category is entirely aggregate OR entirely itemised, and no (person,category) pair mixes both. Summing the main extract gives net expenses (negatives = adjustments/repayments).

## Visualization Strategy (per-view)
- **Rankings** (who spends most) — party-coloured bar leaderboard, rank by total or any category, chamber/party/state/current filters. Hover = exact; click = drill.
- **Explorer** (look up one person) — searchable, sortable table, quarterly sparkline per row.
- **Map** (geography) — Leaflet choropleth of 150 divisions by member total; hover tooltip with member+party+total; click → drill. Senate handled via state totals.
- **Categories** (where money goes) — squarified treemap + party×category per-member heatmap. Treemap click → rank by that category.
- **Parties** — total vs average-per-member bars (average controls for party size). Click → filtered rankings.
- **Trends** — quarterly national bars (2025 election marked) + 5-party line chart.
- **Distribution** — histogram of per-person totals, median line, outlier bins highlighted; House/Senate toggle.
- **Insights** — 8 auto-detected cards (top spender, biggest single claim, travel/car leaders, outliers, per-member party extremes, House-vs-Senate, sharpest quarterly rise).
- **Drill-down** — hash-linkable panel: rank, vs-median, category breakdown, quarterly bars, biggest individual claims (lazy-loaded chunk).

## Test Results
- Tests written: 59 (format, charts geometry, analysis, treemap layout **positions**, svgZoom, pipeline lib CSV+classifiers)
- Tests passed: 59
- Tests failed: 0 (one initial failure — fmtDollars sign placement for negatives — fixed)

## Build Status
- npm install: pass
- npm test: pass (59/59)
- npm run build: pass (206 KB JS / 63 KB gzip incl. Leaflet)
- Local preview: pass

## Deployment
- Repo created: yes (ben-gy/au-mp-expenses)
- GitHub Pages enabled: yes (Actions build_type); CNAME au-mp-expenses.benrichardson.dev set + cycled for cert
- Cloudflare DNS: CNAME au-mp-expenses → ben-gy.github.io created (OK)
- PR created: https://github.com/ben-gy/au-mp-expenses/pull/1
- Deploy workflow: triggered, completed success

## In-browser verification (local preview of production dist)
- All 8 views render; no console errors.
- Drill-down opens on a real trusted click (Albanese → rank #1/334, 8.2× median, category bars, 15 quarterly bars, 12 lazy-loaded line items).
- Map: 150 real polygons, tiles, hover member-join tooltip, legend, 8 state bars.
- About modal opened **while the Map view was active** and confirmed painting above Leaflet (elementFromPoint inside modal; z-index 2200 > Leaflet 1000).
- Treemap: 10 cells, 0 overlaps, 0 bad geometry. Matrix 6 parties × 10 categories.
- Explorer search ("dutton") filters to 1 row.
- Mobile 375px: no horizontal overflow on any of the 8 views or with the drawer open (scrollWidth == clientWidth == 375 everywhere); drawer is full-width.

## Errors & Resolutions
- ABS ASGS CED ArcGIS `query` returned HTTP 400 for all layers → switched to Digital Atlas (ArcGIS Online) FeatureServer for the March 2025 divisions.
- `tsc` failed on unused imports in distribution.ts and an implicit-any from importing pipeline `lib.mjs` in a test → removed unused imports and scoped tsconfig `include` to `src` (tests run under vitest).
- fmtDollars formatted negatives as `$-1,234`; fixed to `-$1,234`.
- Browser-pane `preview_start` picked up a stale root launch.json → ran `vite preview` directly and pointed the pane at localhost:5199.
