# Build Log: Road Crashes (AU)
**Date:** 2026-04-17
**Status:** deployed

## Idea Source
IDEAS.md: "Australian road crash data explorer — standardised accident data across all state road authorities, with heatmaps, trend charts, and filters by contributing factor, road type, LGA, and state."

## Site Details
- **Name:** Road Crashes (AU)
- **Repo:** ben-gy/au-crashes
- **Category:** public-safety
- **Audience:** Road safety researchers, transport planners, policy analysts, journalists, citizens
- **Stack:** Vanilla TypeScript + Vite 6 + Leaflet + Vitest
- **Data strategy:** pipeline (GitHub Actions monthly fetch from data.gov.au)

## Data Sources
- Australian Road Deaths Database (ARDD) Fatalities CSV: https://data.gov.au/data/dataset/australian-road-deaths-database
  - 55,360 individual fatality records
  - Coverage: 1989–2023
  - Fields: state, crash type, road user, gender, age, speed limit, road type, remoteness, LGA, SA4, vehicle involvement, holiday periods
  - Updated monthly by BITRE
  - License: CC BY 3.0 AU

## Architecture Decisions
- **Vanilla TypeScript** over React: single-page tabbed interface with no routing or complex component trees. SVG charts are rendered as template strings, not React components.
- **Pipeline strategy**: The ARDD CSV is 6.6MB raw. Pipeline downloads and processes into optimised JSON (6.1MB column-oriented records + ~20KB aggregated summaries). GitHub Actions runs monthly on the 15th.
- **Column-oriented records format**: Instead of 55K objects with repeated keys (14MB), records are stored as columns + data arrays (6.1MB, ~900KB gzipped). Massive size reduction.
- **Hand-rolled SVG charts**: No D3 or chart library. Bar charts, heatmaps, and sparklines are all generated as SVG template strings. Keeps bundle small (56KB gzipped JS).
- **Leaflet with simplified GeoJSON**: Used simplified polygon coordinates for Australian states rather than full boundary data (which would be 1MB+). Good enough for a state-level choropleth.
- **Light civic theme**: Audience is general public / government — chose light backgrounds with navy (#1e3a5f) and teal accents, not dark/hacker aesthetic.

## Test Results
- Tests written: 38
- Tests passed: 38
- Tests failed: 0
- 2 test suites: utils (29 tests) and glossary (9 tests)
- Covers: formatNumber, formatPercent, sortedEntries, parseRecords, maxValue, heatColor, colour constants, glossary lookups, glossary completeness

## Build Status
- npm install: pass
- npm test: pass (38/38)
- npm run build: pass (189KB JS, 12KB CSS, 56KB gzipped)
- Local preview: pass (HTTP 200)

## Deployment
- Repo created: yes (ben-gy/au-crashes, private)
- GitHub Pages enabled: yes (workflow-based deployment)
- PR created: https://github.com/ben-gy/au-crashes/pull/1
- Workflow triggered: yes (both Deploy and Data Pipeline completed successfully)
- DNS CNAME created: yes (au-crashes.benrichardson.dev → ben-gy.github.io)
- TLS cert: triggered (CNAME cycling applied)

## Errors & Resolutions
1. **Age group normalization bug**: The pipeline's `normAgeGroup` function used `replace(/_/g, '-')` which turned `17_to_25` into `17-to-25` instead of `17-25`. Fixed by using `replace(/_to_/g, '-').replace(/_or_older/, '+')`. This caused the demographics heatmap to show all zeros on first run.
2. **data.gov.au data vintage**: The data on data.gov.au was last updated December 2023 (through October 2023). Attempted to find newer data on the National Road Safety Data Hub (datahub.roadsafety.gov.au) but the CSV download URLs are not publicly predictable. The pipeline can be updated when newer URLs are found.
3. **Preview browser conflict**: The Claude Preview browser was connected to a different server (benrichardson.dev on port 5200). Navigation to localhost:5199 didn't take effect. Verified via curl instead.
