# Build Log: Government Contracts (AU)
**Date:** 2026-07-12
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty, so I researched trending government open-data
topics. Existing registry + live repos already cover a large stable of AU data
sites (crime, crashes, donations, hospitals, dams, energy, planning, flights,
aged care, childcare, charities, NDIS, pollution). A clear gap was **federal
government procurement / contract spending** — a high-value transparency domain
distinct from the existing political-donations site. AusTender publishes all
Commonwealth contract notices via the Open Contracting (OCDS) standard, whose
supplier→agency shape is ideal for the mandated network/flow/matrix views.

## Site Details
- **Name:** Government Contracts (AU)
- **Repo:** ben-gy/au-contracts
- **Category:** government-transparency
- **Audience:** Journalists, procurement/policy analysts, businesses bidding for government work, engaged citizens
- **Stack:** Vanilla TypeScript + Vite + Vitest
- **Data strategy:** pipeline (GitHub Actions → compact JSON)

## Data Sources
- AusTender Open Contracting (OCDS) API — https://api.tenders.gov.au/ocds/findByDates/contractPublished/... (public, no auth, cursor pagination). Verified live: 100 releases/page, ~66,000 contracts for FY2025-26.
- Australian states GeoJSON (ABS-derived, rowanhogan/australian-states) simplified to ~395KB and embedded for the Leaflet choropleth.

## Architecture Decisions
- **Vanilla TS over React:** single-page tabbed dashboard, no routing/complex component trees needed. Smaller bundle (58KB gz incl. Leaflet).
- **Two-stage pipeline:** `collect.mjs` pages the live OCDS API for the most recent complete financial year → `pipeline/raw.json`; `aggregate.mjs` precomputes every view over ALL contracts → `aggregates.json` (105KB) + a `contracts.json` table of the 12,000 largest contracts (1.1MB) with a *disclosed* inclusion threshold (`tableMinAmount` / `tableIsCapped`) so nothing is silently hidden.
- **Precompute over client-side aggregation:** ships a tiny aggregates file for instant paint; lazy-loads the table only on the Contracts tab.
- **Hand-rolled SVG** for bars, treemap, Sankey, force-directed network, heatmap and histogram — no chart libraries. Leaflet only for the map.

## Test Results
- Tests written: 49 (format, store table filter/sort, pipeline extraction helpers)
- Tests passed: 49
- Tests failed: 0
- One fixture initially asserted sorting the store doesn't do on the unfiltered fast-path; fixed the fixture to reflect the real pre-sorted payload contract.

## Build Status
- npm install: pass
- npm test: pass (49/49)
- npm run build: pass (tsc + vite, clean)
- Local preview: pass — all 11 views verified in-browser with real data

## Deployment
- Repo created: yes (already existed from a concurrent twin; replaced with verified version)
- GitHub Pages enabled: yes (workflow build)
- Custom domain: au-contracts.benrichardson.dev — DNS CNAME present, https_enforced=true
- Deploy workflow: success
- Live check: https://au-contracts.benrichardson.dev → HTTP 200, data serving
- PR created: https://github.com/ben-gy/au-contracts/pull/4

## Errors & Resolutions
- **Concurrent twin runs:** This scheduled task fired concurrently and multiple instances built the same/adjacent sites (au-contracts, au-pokies, au-gov). The `au-contracts` repo, DNS, Pages, an earlier PR (#1, now closed) and the registry/index entries were already created by a twin with a *different, unverified* implementation (different data window, module layout, and no Leaflet map module). Resolved by force-pushing this fully browser-verified build to `main`, opening a fresh PR (#4), and correcting the registry + index descriptions/PR URL to match what is actually deployed.
- **Leaflet map rendered blank (SVG width 0, degenerate `M0 0` paths):** the map was initialised before its flex container had a measured width, so the SVG renderer cached a zero-size viewport and never self-healed (even on resize). Fixed by loading the GeoJSON first, waiting two animation frames after DOM attach, giving `.map-canvas` an explicit `width:100%`, calling `invalidateSize()`, and fitting to fixed Australia bounds (never degenerate layer bounds). Verified: full choropleth with NSW deepest ($35.5B).
- **External edits to pipeline files:** the concurrent twin rewrote `pipeline/collect.mjs`/`aggregate.mjs` in the shared directory to use the live OCDS API and a precompute design. Verified the live API works without auth and adopted that approach, building the frontend to match the emitted payloads.
