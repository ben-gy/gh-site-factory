# Build Log: Pollution Watch (AU)
**Date:** 2026-06-23
**Status:** build_complete · DNS_complete · deploy_blocked_by_gh_auth

## Idea Source
Researched. IDEAS.md was empty. Searched data.gov.au for unique high-utility government datasets and confirmed the [National Pollutant Inventory](https://data.gov.au/data/dataset/043f58e0-a188-4458-b61c-04e5b540aea4) was both rich (8,140 facilities × 93 substances × 25 years × air/water/land × off-site transfers × ANZSIC industry classification) and unique — no Australian site currently presents the NPI as a searchable consumer-facing dashboard.

## Site Details
- **Name:** Pollution Watch (AU)
- **Repo:** ben-gy/au-pollution (not yet created — gh auth blocked)
- **Custom domain:** https://au-pollution.benrichardson.dev
- **Category:** environment
- **Audience:** Australian residents worried about pollution near them, journalists, public health advocates, students.
- **Stack:** Vanilla TypeScript + Vite 6 + Leaflet 1.9 (no chart library — all SVG hand-rolled)
- **Data strategy:** Pipeline. emissions.csv is 276 MB so the pipeline streams it line-by-line and aggregates to ~9 MB of JSON.

## Data Sources
| Source | URL |
|--------|-----|
| NPI facilities/emissions/transfers/substances/ANZSIC | https://data.gov.au/data/dataset/043f58e0-a188-4458-b61c-04e5b540aea4 |
| ABS Estimated Resident Population June 2024 | abs.gov.au (embedded as static JSON) |

## Architecture Decisions
- **Vanilla TS over React** — single-page tool with view tabs; no nested-component complexity warranted a framework.
- **Pipeline over runtime-fetch** — emissions.csv is 276 MB; impossible to ship to browsers. Aggregation runs in CI and the browser only sees ~9 MB of pre-computed JSON.
- **Streaming CSV parser, not PapaParse** — keeps the pipeline dependency-free and peak memory under 200 MB.
- **Hand-rolled SVG charts** — bar charts, treemap (squarified), Sankey-style flow, stacked bars, heatmap, year-by-year history. No D3 / Recharts dependency.
- **Leaflet with `preferCanvas: true`** — 1,500-marker map performant on mobile.
- **Light/civic theme** — earth tones with teal accent (rather than dark/hacker), to match the audience (residents, journalists) and the subject (environment/health).

## Views Built (9 total, well above the 5-view minimum)
1. Overview — KPI cards, top-10, medium breakdown, per-capita ranking, persistent emitters
2. Map (Leaflet) — every facility, scaled circles by lifetime emission, substance/year filter
3. Search — debounced free-text search across name/business/suburb/postcode/ABN with state and reporting-year filters
4. Substances — 93 substances grouped by category; click for trend chart, state breakdown, top-25 emitters
5. Industries — squarified treemap, division leaderboard, top-20 subdivisions
6. Flow — Sankey-style ribbon diagram of industry → substance category
7. States — raw vs per-capita comparison + state × year heatmap
8. Trends — national stacked bar chart by medium + year-over-year change table
9. Insights — auto-detected anomalies (biggest jumps/reductions, persistent top-10 across 5 years)
   plus a facility-detail slide-in panel reachable from any view.

## Test Results
- Tests written: 57 (across `tests/format.test.ts`, `tests/analysis.test.ts`, `tests/colors.test.ts`)
- Tests passing: 57 / 57
- Coverage: formatters, mass formatting (kg → t → kt → Mt), per-capita math, search scoring/filtering, state grouping, substance trend direction, anomaly detection, rolling change windows, color palette stability.

## Build Status
- npm install: ✅
- npm test: ✅ (57/57)
- npm run build: ✅ (213 KB JS / 32.6 KB CSS / 8 MB data, gzipped to ~63 KB JS + 10 KB CSS)
- Local preview: ✅ (verified all 9 views and mobile responsive layout)

## Visual Verification (via Chrome MCP preview)
- ✅ Overview renders with correct KPIs (4,552 reporting / 93 substances / 101.88 Mt lifetime)
- ✅ Map shows top-1500 facilities across Australia with realistic geographic distribution
- ✅ Substances categorised (Criteria pollutants, Particulates, Heavy metals, VOCs, …) with drill-down
- ✅ Industries treemap renders Manufacturing/Mining/Electricity dominating
- ✅ Flow Sankey diagram renders ribbons from divisions → substance categories
- ✅ States view shows QLD biggest in raw tonnes, WA biggest per-capita
- ✅ Trends shows 25-year stacked national chart
- ✅ Insights auto-detects 12+ year-over-year anomaly cards
- ✅ Facility detail slide-in panel works (clicked Mt Isa Mines → 7.56 Mt lifetime air)
- ✅ Mobile (375px) — tabs scroll horizontally on row 2, KPI cards stack, lede heading sized appropriately
- ✅ Light theme correct for civic audience
- ✅ No console errors

## Deployment
- Local git init + commit: ✅
- gh repo create: ❌ HTTP 401 — gh auth token invalid (`gh auth login` needed)
- Cloudflare DNS CNAME au-pollution → ben-gy.github.io: ✅ (created)
- GitHub Pages enable: ⏸ blocked by missing repo
- PR creation: ⏸ blocked by missing repo
- TLS cert: ⏸ pending repo + Pages + CNAME

## Manual deploy steps required
1. `gh auth login` (or set GH_TOKEN env var)
2. `cd /Users/benrichardson/Code/gh-site-factory/sites/2026-06-23-au-pollution && gh repo create ben-gy/au-pollution --private --source=. --push`
3. `gh api repos/ben-gy/au-pollution/pages -X POST -f build_type=workflow`
4. `gh api repos/ben-gy/au-pollution/pages -X PUT -f cname=au-pollution.benrichardson.dev`
5. Wait for `Deploy to GitHub Pages` Action to finish
6. Re-add to `index/sites.json` and `index/sites.txt` once live (entry below)

### sites.json entry (ready)
```json
{
  "name": "Pollution Watch (AU)",
  "slug": "au-pollution",
  "description": "Every industrial polluter in Australia — 8,140 facilities, 93 substances, 25 years of National Pollutant Inventory data, with auto-detected anomalies.",
  "url": "https://au-pollution.benrichardson.dev",
  "repo": "https://github.com/ben-gy/au-pollution",
  "tags": ["environment", "pollution", "transparency"],
  "type": "web",
  "date": "2026-06-23"
}
```

## Errors & Resolutions
- **Map canvas had width 0 on first render** → added `requestAnimationFrame(() => map.invalidateSize())` and explicit `width: 100%` on `.map-container`.
- **Mobile lede h1 stayed 36px despite media query** → media query was placed before the base rule. Moved all mobile media queries to the end of `styles.css` (and added `min-width: 100%; flex-basis: 100%` on `.tabs` so the tab strip reliably wraps onto its own row).
- **gh repo create / push** → HTTP 401 (token invalid). Two previous sites (au-ndis-watch, au-agedcare) hit the same wall — local user needs to re-run `gh auth login`.
