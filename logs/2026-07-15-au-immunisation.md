# Build Log: Immunisation Coverage
**Date:** 2026-07-15
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty. The catalog is heavy on AU government data explorers, so I looked for a fresh, geographic, distinct dataset with real ongoing utility. Considered ACARA/MySchool (NAPLAN bulk data requires a Data Access Program application — not cleanly open, rejected), PBS Item Report (data.gov.au copy stops at 2016 — too stale, rejected), Medicare bulk billing (viable), and childhood immunisation coverage. Chose **childhood immunisation coverage**: fresh (quarterly, rolling four quarters to Sep 2025), openly downloadable, finely geographic (SA3), with a clear 95% herd-immunity threshold and a First Nations equity angle. No existing site covers it.

## Site Details
- **Name:** Immunisation Coverage
- **Repo:** ben-gy/au-immunisation
- **Category:** health (index: data-explorers, country AU)
- **Audience:** Parents checking their local area; GPs / public-health staff; journalists & policy people.
- **Stack:** Vanilla TypeScript + Vite + Vitest; Leaflet for the SA3 choropleth; hand-rolled SVG for all charts.
- **Data strategy:** pipeline. Cron `23 4 6 1,4,7,10 *` — QUARTERLY, matching the AIR rolling-four-quarters publication cadence; staggered day 6 / minute 23.

## Data Sources
- AIR childhood immunisation coverage (PHN all children, PHN First Nations, SA3 per-state) — https://www.health.gov.au/resources/collections/childhood-immunisation-coverage-data-phn-and-sa3
- ABS ASGS 2021 SA3 boundaries (generalised layer) — https://geo.abs.gov.au/arcgis/rest/services/ASGS2021/SA3/MapServer

## Architecture Decisions
- **Vanilla TS** (default) — single-page multi-view tool, no need for React.
- **Pipeline, content-based file discovery:** the department re-publishes each quarter under inconsistent filenames. `collect.mjs` scrapes the collection + publication pages, ranks candidate .xlsx by their `/files/YYYY-MM/` folder date, downloads and *validates by content* (sheet title + header) to fill each slot (SA3 per-state, PHN all-children, PHN First Nations). This should survive future quarterly re-publishes.
- **Boundaries:** ABS SA3_GEN fetched as GeoJSON (paginated), simplified with mapshaper to 1.3% → 882 KB (raw was 123 MB). Real coastlines, in the 100 KB–1 MB target.
- **Normalisation gotchas handled:** PHN workbooks give proportions (0–1), SA3 give percents (0–100) — auto-detected per file. Suppression tokens (NP/NM/≥95.00/≥99.00) and `0`-means-not-assessed-at-this-age both mapped to null. Age labels normalised across "1 Year olds" and older "12 months" forms so the 2015–2025 trend parses.
- **Views chosen for the data's shape:** map / leaderboard / explorer / vaccine×state matrix / First Nations equity / trend / distribution / insights. No Sankey or network — the data has no inter-entity flows.

## Test Results
- Tests written: 46 (format incl. positional histogram-layout, scale, data analysis)
- Tests passed: 46
- Tests failed: 0

## Build Status
- npm install: pass
- npm test: pass (46/46)
- npm run build: pass (190 KB JS / 33 KB CSS)
- Local preview: pass — all 8 views + drilldown verified, no console errors, no mobile overflow at 375px (asserted per view + drilldown)

## Deployment
- Repo created: yes (ben-gy/au-immunisation)
- GitHub Pages enabled: yes (workflow build)
- Cloudflare DNS: created (au-immunisation → ben-gy.github.io)
- TLS: HTTPS serving 200 (https_enforced flag lagging but cert live)
- PR created: https://github.com/ben-gy/au-immunisation/pull/1
- Deploy workflow: success; live bundle matches local dist (index-BwTK-3ir.js)
- Production verified: all 8 views render, drilldown opens via real map click and hash, zero console errors on the live URL.

## Errors & Resolutions
- **Trend only got 4 years initially:** older PHN workbooks label ages in months ("12 months") not years, so `ageKey` returned "12" not "1". Fixed the normaliser → 10 annual points (2015–2025, 2016 unavailable).
- **sa3.geojson was 5 MB** at 8% simplify (SA3_GEN is high-detail). Re-simplified to 1.3% + precision 0.001 → 882 KB.
- **Data Pipeline CI failed on `npm ci`:** mapshaper's transitive `@types/node` drifted the lockfile out of sync. Switched the pipeline workflow to `npm install` (robust for a scraper) and regenerated the lockfile; committed the fix.
