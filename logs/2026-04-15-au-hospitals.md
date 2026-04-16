# Build Log: AU Hospital Watch
**Date:** 2026-04-15
**Status:** deployed

## Idea Source
IDEAS.md: "Australian hospital & ED performance explorer — standardised waiting times, elective surgery waits, and bed occupancy across all states and territories, comparable by hospital, LGA, and state. States currently publish this in inconsistent formats and timeframes."

## Site Details
- **Name:** AU Hospital Watch
- **Repo:** ben-gy/au-hospitals
- **Category:** health
- **Audience:** Australian patients, carers, and health policy researchers comparing hospital ED and elective surgery performance
- **Stack:** Vanilla TypeScript + Vite 6
- **Data strategy:** runtime-fetch (AIHW MyHospitals API with 24hr localStorage cache)

## Data Sources
- AIHW MyHospitals API: https://myhospitalsapi.aihw.gov.au/api/v1
  - `/reporting-units?reporting_unit_type_code=H` — 1,427 hospitals
  - `/flat-data-extract/MYH-ED-TIME?measure_code=MYH0005&start_date=2024-07-01` — ED 4-hour departure rate (~2,344 hospital-level records)
  - `/flat-data-extract/MYH-ED-TIME?measure_code=MYH0013` — 90th percentile ED time (879 records)
  - `/flat-data-extract/MYH-ED-TIME?measure_code=MYH0036` — Median ED time (879 records)
  - `/flat-data-extract/MYH-ED-TIME?measure_code=MYH0012` — Total presentations
  - `/flat-data-extract/MYH-ED-WAITS?measure_code=MYH0010` — Triage on-time % (1,363 records)
  - `/flat-data-extract/MYH-ES?reporting_unit_code=HXXXX` — Elective surgery (lazy, per hospital)

## Architecture Decisions
- **Vanilla TypeScript** (not React): single-page tool with no complex state management trees; vanilla TS + DOM was simpler
- **Runtime-fetch**: AIHW API is free, no auth, and data is small enough (5 parallel requests on load, ~5,000 records total) to fetch at runtime; localStorage caching avoids repeat API hits
- **No Tailwind**: vanilla CSS with custom properties — cleaner and smaller bundle for a single-page tool
- **Light civic theme**: audience is general public (patients, carers), not technical users; used health blue accent (#0369a1) with white backgrounds
- **Lazy elective surgery load**: elective surgery data has 100+ records per hospital across specialty categories; loaded per-hospital on demand rather than upfront for all 1,427 hospitals

## Test Results
- Tests written: 52
- Tests passed: 52
- Tests failed: 0
- Coverage: all utility functions (formatters, class generators, normalisers, escape helpers)

## Build Status
- npm install: pass
- npm test: pass (52/52)
- npm run build: pass (33 kB JS, 16 kB CSS)
- Local preview (HTTP 200): pass
- DOM verification: 1,165 hospitals loaded, 5 state cards, 20 ranking rows, 0 errors

## Deployment
- Repo created: yes — https://github.com/ben-gy/au-hospitals
- GitHub Pages enabled: yes (workflow-based)
- Cloudflare DNS CNAME created: au-hospitals → ben-gy.github.io (DNS only)
- TLS cert issued: yes (https_enforced=true confirmed)
- PR created: https://github.com/ben-gy/au-hospitals/pull/3

## Errors & Resolutions (initial build)
1. TypeScript unused import/param errors — resolved
2. Cloudflare cert required CNAME cycling — resolved on attempt 6

## V2 Redesign (same day)
User feedback identified 5 issues:
1. Victoria + QLD + TAS missing — API returns mixed-case state codes (Vic, Qld, Tas), code expected uppercase. Fixed with `.toUpperCase()` in api.ts
2. Not map-based — replaced table layout with Leaflet map as primary interface. Markers colored by 4-hr rate, sized by presentation volume (log scale)
3. Rankings misleading — removed top/bottom 10 (biased by hospital size). Map marker sizing naturally communicates volume vs performance
4. No clear user value — added "Near me" geolocation with distance sorting, state filter with map fly-to
5. No context — added info modal with data explanation, methodology, caveats about rural vs urban bias, and metric definitions

**Files changed:** 16 (1,342 insertions, 1,278 deletions)
**Tests:** 63 passing (added haversineKm, markerRadius, formatDistance tests)
**Bundle:** 31 kB app JS + 150 kB Leaflet (code-split) + 32 kB CSS
