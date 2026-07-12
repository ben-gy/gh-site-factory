# Build Log: Pokies Losses (NSW)
**Date:** 2026-07-12
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty. After discovering that many candidate domains
(aged care, childcare, charities, NDIS, air pollution, road deaths) already exist
as `ben-gy` repos not listed in registry.json, I researched an uncovered, high-impact
Australian open-data topic: **poker machine (pokies) gambling losses by council area**.
NSW has more pokies than any other state (~88,000 machines) and publishes rich
per-LGA gaming-machine data. WebSearch + WebFetch confirmed clean, downloadable
XLSX reports from Liquor & Gaming NSW. Victoria was evaluated too but its VGCCC
files sit behind bot protection (return HTML to automated fetches), so the product
was scoped to NSW where the data downloads cleanly and completely.

## Site Details
- **Name:** Pokies Losses (NSW)
- **Repo:** ben-gy/nsw-pokies
- **Category:** public-health / government-transparency
- **Audience:** NSW residents checking their suburb, journalists, councillors, gambling-harm researchers
- **Stack:** Vanilla TypeScript + Vite + Vitest; Leaflet for the map; hand-rolled SVG for all other charts
- **Data strategy:** pipeline (committed generated JSON so the site works immediately; weekly cron refresh)

## Data Sources
- Liquor & Gaming NSW — Gaming Machine Data (Clubs, by LGA): https://www.nsw.gov.au/business-and-economy/liquor-and-gaming/gaming/gaming-machine-data-reports
- Liquor & Gaming NSW — Gaming Machine Data (Hotels, by LGA): same portal
- Geoscape Administrative Boundaries — NSW LGA (GeoJSON): https://data.gov.au/data/dataset/nsw-local-government-areas

20 XLSX reports (clubs + hotels, FY annual for 2023-24 & 2024-25 plus bi-annual
halves back to 2019) were parsed and aggregated. Statewide FY2024-25 total:
$9.06bn player losses ($5.13bn clubs + $3.93bn hotels), 87,855 machines.

## Architecture Decisions
- **Vanilla TS over React:** single-page tabbed dashboard, no routing/component tree needed. Smaller bundle (57KB gz JS).
- **Pipeline, committed data:** the NSW files are behind a normal web server (download cleanly with a browser UA), but XLSX parsing and boundary simplification are best done once in CI. `collect.mjs` downloads + simplifies (mapshaper); `aggregate.mjs` parses (xlsx) + joins clubs/hotels + computes per-adult/per-machine + builds trend + auto-insights. Output JSON committed so the first deploy has data.
- **NSW-only:** VIC data is bot-blocked; NSW alone is the pokies capital and yields a complete, accurate dataset. Naming uses the "(NSW)" region-code convention.
- **Per-adult metric:** the annual reports include LGA population; adults estimated at 79% (ABS NSW 18+ share). Clubs and hotels are occasionally grouped differently by L&GNSW — combined rows merge on matching area labels, single-sector rows are badged.
- **Map:** Leaflet + real simplified GeoJSON (34MB → 1.4MB via mapshaper dissolve+simplify). All 129 data LGAs matched to boundary features.

## Test Results
- Tests written: 43 (format, analysis, charts/squarify)
- Tests passed: 43
- Tests failed: 0

## Build Status
- npm install: pass
- npm test: pass (43/43)
- npm run build: pass (186KB JS / 57KB gz, 35KB CSS / 11KB gz)
- Local preview: pass — verified all 7 views + drill-down + mobile + dark-mode in-browser; no console errors
- Full pipeline (collect + aggregate): pass end-to-end (20 files downloaded, boundaries simplified, 0 failures)

## Deployment
- Repo created: yes (ben-gy/nsw-pokies)
- GitHub Pages enabled: yes (workflow build)
- Custom domain: nsw-pokies.benrichardson.dev — Cloudflare CNAME created, Pages CNAME set + cycled
- Live: yes — https://nsw-pokies.benrichardson.dev returns 200 over HTTPS; data endpoints serve
- Deploy workflow: completed success
- TLS: cert issued (HTTPS serving 200); `https_enforced` was still flipping to true at hand-off (normal for a fresh domain, finalises within minutes)
- PR created: https://github.com/ben-gy/nsw-pokies/pull/2

## Errors & Resolutions
- **VIC data bot-blocked:** vgccc.vic.gov.au / data.vic files return HTML to automated fetches. Resolution: scoped the product to NSW (complete, cleanly downloadable, and the largest pokies jurisdiction).
- **Map container collapsing to 0px:** `.map-canvas { max-height: 74vh }` clamped the fixed height to 0 when the measurement context reported 0 viewport height. Resolution: replaced with `min-height: 460px` and switched the fit logic to a ResizeObserver that fits once the container has real layout height. Verified full NSW choropleth renders.
- **Header-row detection grabbed the title row:** the report title also contains "Local Government Area". Resolution: require the header row's second column to be "Net Profit".
```
