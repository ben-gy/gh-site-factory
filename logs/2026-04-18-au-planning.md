# Build Log: Development Applications (AU)
**Date:** 2026-04-18
**Status:** deployed

## Idea Source
IDEAS.md: "Australian planning & development applications map — unified DA/planning permit explorer across all councils, standardised by application type, value, status, and applicant. Currently every council has its own portal with no consistent format."

## Site Details
- **Name:** Development Applications (AU)
- **Repo:** ben-gy/au-planning
- **Category:** housing
- **Audience:** Homebuyers researching suburbs, residents monitoring local development, planning researchers comparing council activity
- **Stack:** Vanilla TypeScript + Vite 6 + Vitest + Leaflet (maps) + hand-rolled SVG (charts)
- **Data strategy:** pipeline (GitHub Actions fetches from council open data APIs every 6 hours)

## Data Sources
- **City of Casey (VIC)** — OpenDataSoft API at `data.casey.vic.gov.au`, 23,261 total records (5,000 fetched for initial build), no auth required, CC BY 3.0 license
- **PlanningAlerts API** — Infrastructure in place but requires API key as GitHub secret (covers 212 authorities, 89% of Australia)

## Architecture Decisions
- **Vanilla TypeScript** chosen over React — single-page data tool with tab-based views, no complex routing or component trees needed. Simpler, smaller bundle.
- **Pipeline + embedded data** — Pipeline fetches from council APIs and commits JSON to `public/data/`. Initial dataset is real data from City of Casey, not synthetic.
- **Hand-rolled SVG** for all charts (horizontal bars, timeline, Sankey flow, histogram) — no chart library dependency, keeping bundle at 209KB (61KB gzipped).
- **Leaflet** for map view with suburb-level circle markers — individual address geocoding would require a geocoding API, so suburb centroids are used instead.
- **Light theme** — civic/government audience, not a technical monitoring tool. Navy (#1e3a5f) and teal (#0d9488) accent palette.
- **Keyword-based category classification** — DA descriptions are classified into 13 categories (Residential, Subdivision, Commercial, etc.) using regex matching in `pipeline/collect.mjs`. Simple and effective for the dataset.

## Test Results
- Tests written: 62
- Tests passed: 62
- Tests failed: 0
- Test suites: 3 (utils, glossary, data filtering)

## Build Status
- npm install: pass
- npm test: pass (62/62)
- npm run build: pass (209KB JS, 61KB gzipped)
- Local preview: pass (HTTP 200, all data files load)

## Deployment
- Repo created: yes (ben-gy/au-planning, private)
- GitHub Pages enabled: yes (workflow-based)
- PR created: https://github.com/ben-gy/au-planning/pull/1
- Workflow triggered: yes (completed successfully)
- DNS record created: yes (CNAME au-planning → ben-gy.github.io in Cloudflare)
- TLS cert: yes (https_enforced=true after cert cycle)

## Views Implemented (7)
1. **Map** — Leaflet with suburb circle markers sized by DA count, click popups with stats
2. **Table** — Sortable by date/suburb/category/decision/processing time, 100-row paginated
3. **Categories** — Horizontal bars by DA type + decision outcome breakdown
4. **Timeline** — Monthly volume chart + annual totals + top months
5. **Flow** — Sankey diagram: application category → decision outcome
6. **Suburbs** — Top suburbs bar chart + detail table with approval rates + approval rate ranking
7. **Processing** — Histogram of processing times + percentile table + speed bucket breakdown

## Key Data Insights (from City of Casey)
- 82.0% approval rate, 6.3% refusal rate
- Top category: Residential (1,787 of 5,000)
- Most active suburb: Berwick (574 applications)
- Data spans 2021-07 to 2026-04

## Errors & Resolutions
- TypeScript error in processing.ts: `.filter()` on tuple array lost type information. Fixed by using intermediate variable with explicit type annotation.
- Unused import `sortedEntries` in suburbs.ts. Removed.
- No other errors encountered.
