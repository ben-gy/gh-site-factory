# Build Log: Road Deaths (AU)
**Date:** 2026-04-17
**Status:** deployed

## Idea Source
IDEAS.md: "Australian road crash data explorer — standardised accident data across all state road authorities, with heatmaps, trend charts, and filters by contributing factor, road type, LGA, and state."

## Site Details
- **Name:** Road Deaths (AU)
- **Repo:** ben-gy/au-road-deaths
- **Category:** public-safety
- **Audience:** Road safety researchers, journalists, policy analysts, students
- **Stack:** Vanilla TypeScript + Vite 6 + Vitest + Leaflet
- **Data strategy:** pipeline (BITRE XLSX → JSON)

## Data Sources
- BITRE Australian Road Deaths Database (https://datahub.roadsafety.gov.au/progress-reporting/monthly-road-deaths)
  - bitre_fatalities_feb2026.xlsx — 58,389 records, 1989–Feb 2026
  - Downloaded from datahub.roadsafety.gov.au
- ABS estimated resident population (embedded, 2024 estimates)

## Architecture Decisions
- Vanilla TypeScript (no React) — single-page tab-switching app, no complex component trees needed
- Hand-rolled SVG charts — avoids D3/Recharts bundle size, full control over interactivity
- Leaflet for map — choropleth with simplified GeoJSON state boundaries
- Pipeline strategy — BITRE updates monthly; GitHub Actions downloads and processes XLSX on the 15th
- Data split: aggregates.json (20KB) + records.json (3MB) — aggregates load instantly, records lazy-load for the table view
- Used BITRE Feb 2026 XLSX instead of data.gov.au CSV (data.gov.au copy is Oct 2023, ~3 years behind)

## Test Results
- Tests written: 33
- Tests passed: 33
- Tests failed: 0

## Build Status
- npm install: pass
- npm test: pass (33/33)
- npm run build: pass (193KB JS, 13KB CSS)
- Local preview: pass (HTTP 200, data JSON served correctly)

## Deployment
- Repo created: yes (ben-gy/au-road-deaths)
- GitHub Pages enabled: yes (workflow build type)
- PR created: https://github.com/ben-gy/au-road-deaths/pull/3
- Workflow triggered: yes (completed successfully)
- DNS CNAME created: yes (au-road-deaths.benrichardson.dev → ben-gy.github.io)
- TLS cert: provisioning (CNAME cycled, DNS resolving correctly)

## Errors & Resolutions
- State abbreviation mismatch: BITRE Feb 2026 XLSX uses uppercase state codes (QLD, VIC, TAS) while initial code used mixed case (Qld, Vic, Tas). Fixed by updating STATES constant and GeoJSON to use uppercase.
- Note: concurrent run created similar "au-crashes" repo from same IDEAS.md entry. Both coexist — au-road-deaths uses newer Feb 2026 data (58K records) vs au-crashes Oct 2023 data (55K records).
