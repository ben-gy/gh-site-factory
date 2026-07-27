# Build Log: Population Change
**Date:** 2026-07-27
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty. The catalog is saturated with AU government/economy
data explorers, so idea selection was mostly about finding a genuinely-new, well-shaped,
CURRENT, cleanly-accessible dataset. Ruled out several candidates during research:
- **Greenhouse-gas / Safeguard-Mechanism emissions** — already logged in EXPANSION_IDEAS.md
  (2026-07-22) as an au-pollution expansion, rejected as a standalone under the expansion
  rule (same facility-map/treemap/rankings shape as au-pollution).
- **Cancer (AIHW CDiA)** — the current workbooks are hard-gated behind a JS "data and
  resources" component (curl/API probing hit 404s; the Browser pane timed out on aihw.gov.au),
  and the only clean machine-readable copy (data.gov.au ACIM CSVs) ends in 2011 with no
  survival data — too dated for a marquee product.
- **GrantConnect grants** — not cleanly on data.gov.au (matches were research-grant reports).
- **Disaster declarations** — only per-event LGA shapefiles, no unified register.
- **ACCC product recalls** — not a clean CKAN dataset.

Chosen: **Population Change** — ABS Regional Population (ERP + components of change by LGA).
A genuinely new topic (no existing site is about population growth/decline and its drivers),
current, richly geographic, and on the fleet's most-proven data source (ABS SDMX + ASGS
boundaries).

## Site Details
- **Name:** Population Change
- **Repo:** ben-gy/au-population
- **Category:** demographics
- **Audience:** Australians checking their town's trajectory; journalists chasing fastest-growing/
  emptying places; planners and demographers.
- **Stack:** Vanilla TypeScript + Vite + Vitest, Leaflet
- **Data strategy:** pipeline — annual cron (`29 6 7 4 *`, staggered). ABS publishes Regional
  Population once a year (Mar/Apr); yearly is the correct proportional cadence. Embeds the ABS
  reference year, not a run timestamp, so re-runs are idempotent.

## Data Sources
- ABS Regional Population — ERP by LGA, `ABS_ANNUAL_ERP_LGA2024` (2001→2024) — data.api.abs.gov.au SDMX
- ABS Regional Population — components of change, `ERP_COMP_LGA2024` (2022-2024) + `ERP_COMP_LGA2021`
  (2017-2021), stitched — data.api.abs.gov.au SDMX
- ABS ASGS 2024 LGA boundaries — geo.abs.gov.au ArcGIS LGA_GEN layer, mapshaper-simplified

## Architecture Decisions
- Vanilla TS (single-page tabbed tool, one map) over React — smaller bundle, simpler.
- Dependency-free `pipeline/parse.mjs` (own RFC-4180 SDMX-CSV reader + all shaping) shared with the
  46-test suite, so the exact pipeline code is unit-tested. mapshaper is the only pipeline dep.
- Long ERP series (24 yrs) for the growth trajectory; components (8 yrs, stitched across two LGA
  editions keeping only 2024-boundary codes) for the engine decomposition.
- National components trend summed from the kept LGAs (the AUS row only exists 2022-2024),
  cross-checked against the published AUS total.
- 547 LGAs; population.json 1.19 MB (full per-LGA series + component series), lga.geojson 851 KB.

## Per-view UX critique (recorded per the standards)
- **Overview** — Q: is Australia growing, and how? Hover: engine bars, type segments. Click: tiles/
  movers → drawer, type legend → Explorer filtered by type. Next: rankings link.
- **Growth Engines (signature)** — Q: what drives each place? Quadrant scatter, hover shows all three
  rates + growth, click → drawer, state filter, zoom/pan. Decomposition bars: hover each engine seg,
  click → drawer.
- **Map** — Q: where? 5 measures, hover tooltip per polygon, click → per-council drawer above Leaflet.
- **Rankings** — Q: who's top/bottom? metric + state + direction; median stated; row → drawer.
- **Trends** — Q: how did it change? 3-engine lines with COVID band; per-state population lines; hover dots.
- **Treemap** — Q: big picture? nested state→LGA, hover exact values, click → drawer.
- **States matrix** — Q: which state grows by which engine? heatmap + composition bars; row → Explorer.
- **Explorer** — look up any council; sortable, filterable, sparklines, engine bars, histogram → filter.
- **Insights** — auto findings; cards that name a council are clickable → drawer.

## Test Results
- Tests written: 46 (parse: 13, analysis: 9, layout: 13, render: 11)
- Tests passed: 46
- Tests failed: 0
- Notable: a headless jsdom **render smoke test** renders every content view + a sample of every
  demographic-type drawer + the About modal against the REAL built `population.json`, asserting no
  NaN/undefined/errors, treemap cells in-bounds, and populated histogram+table. This substitutes for
  the live-browser click-through that the environment blocked.

## Build Status
- npm install: pass
- npm test: pass (46/46)
- npm run build: pass (JS 218 KB / 66 KB gzip)
- Local preview: served 200 (index + data), but in-app Browser pane and Claude-in-Chrome both
  unavailable for localhost, so no in-browser click-through.

## Reconciliation gate
Two INDEPENDENT ABS series must agree, or the build fails:
1. long annual ERP == component-file ERP (POP_COMP=10) for every LGA — 547/547 checked, 0 mismatched.
2. net internal migration sums to ~0 nationally — sums to exactly 0.
Both pass.

## Deployment
- Repo created: yes (ben-gy/au-population)
- GitHub Pages enabled: yes (workflow build)
- Deploy workflow: green on first run
- Custom domain: DNS CNAME created in Cloudflare; Pages CNAME set + cycled for cert
- Production: HTTPS 200 at https://au-population.benrichardson.dev; github.io 301→custom domain;
  live bundle index-D2eDS_ry.js matches local dist byte-for-byte; data + og.png all 200
- https_enforced: still false at hand-over (cert freshly issued; HTTPS already 200) — re-check
- PR created: https://github.com/ben-gy/au-population/pull/1

## Errors & Resolutions
- Build tsc initially typechecked the tests dir and failed on the `.mjs` import + implicit any →
  scoped `tsconfig.include` to `src` (fleet convention); vitest still runs tests.
- Copied THIRD-PARTY-NOTICES + served notices carried au-vehicles data lines → corrected to ABS
  Regional Population + ASGS 2024.
- One test asserted natRate to 3dp but parse rounds to 2dp → fixed the assertion.

## Limitations
- Live-URL in-browser real-click verification could not be run (Browser pane blocks the domain by
  policy; Claude-in-Chrome not connected). Mitigated by the headless render test against the
  byte-identical built data. Re-run production real-click verification + confirm https_enforced flipped.
