# Build Log: Government Contracts (AU)
**Date:** 2026-07-12
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty. Reviewed registry.json and the full `ben-gy` repo list to find an uncovered domain. Existing AU data sites already cover flights, crime, dams, donations, hospitals, planning, crashes/road-deaths, aged care, charities, childcare, NDIS, pollution, energy, and pokies. **Government procurement / contracts spending (AusTender)** was uncovered, highly topical (consulting scandals), searchable, and distinct from political donations (au-donations) — it tracks where public money goes to suppliers, not who gives money to parties. Verified data availability via WebSearch and the AusTender OCDS API before committing.

## Site Details
- **Name:** Government Contracts (AU)
- **Repo:** ben-gy/au-contracts
- **Category:** government-transparency
- **Audience:** journalists, policy researchers, staffers, procurement/bid managers, taxpayers
- **Stack:** Vanilla TypeScript + Vite + Vitest
- **Data strategy:** pipeline (OCDS API → NDJSON → aggregated JSON in public/data/)

## Data Sources
- AusTender OCDS API — https://api.tenders.gov.au/ocds/findByDates/contractPublished/{start}/{end} (cursor-paginated, 100 releases/page)
- Department of Finance — Australian Government contract notices (CC BY 3.0 AU)
- UNSPSC top-level segment map (embedded) for category names

## Architecture Decisions
- **Pipeline over embedded/runtime-fetch:** the full dataset (166k raw notices) is far too large to embed or fetch at runtime. `pipeline/collect.mjs` pages the OCDS API month-by-month with concurrency (6), extracting minimal fields to NDJSON; `pipeline/aggregate.mjs` de-duplicates by OCID (latest amendment wins) and rolls up into compact JSON. The browser fetches only aggregates (largest file: suppliers.json ~6MB, ~1.3MB gzipped).
- **Vanilla TS over React:** single-page tabbed dashboard; no routing/component-tree complexity needed.
- **Hand-rolled SVG for all charts** (bars, treemap, force-directed network, Sankey, matrix heatmap, monthly line) — no chart library. Leaflet only for the supplier-state map (proportional circle markers on a CARTO basemap; no hand-drawn SVG paths).
- **Two full financial years (2023-24, 2024-25)** collected so year-on-year comparison and seasonality work.
- **"Professional & consulting" framed honestly** as a grouping of whole UNSPSC categories, broader than AusTender's narrow consultancy flag — noted in copy, glossary and About modal to avoid overstating.
- **Big-firm ranking canonicalised** (regex word-boundary matching → canonical labels, aggregated across all categories) after an initial naive version produced duplicate/false-positive noise.

## Data Snapshot
- Period: 2023-07-01 → 2025-06-30
- 165,623 raw releases → 121,029 unique priced contracts
- Total value: $126.76B · 34,102 suppliers · 131 agencies
- Top agency: Department of Defence ($67.4B). Top supplier: Reserve Bank FMS account ($15.3B).
- Professional & consulting grouping: $53.9B (42.5%). Big firms: Accenture $876M, IBM $763M, DXC $428M, Deloitte $388M, KPMG $361M, EY $326M; PwC just $2.6M (post-tax-leaks-scandal collapse).

## Test Results
- Tests written: 71 (format: 31, analysis: 24, views smoke + content: 16)
- Tests passed: 71
- Tests failed: 0
- The views suite mounts all 11 view renderers against a mock dataset (jsdom) and asserts key content, plus search-filter behaviour.

## Build Status
- npm install: pass
- npm test: pass (71/71)
- npm run build: pass (strict TS + Vite)
- Local preview: pass (HTTP 200 on index + all /data/*.json endpoints; valid meta)

## Deployment
- Repo created: yes (repo already existed empty from an earlier/sibling incarnation created 06:35Z; pushed main directly)
- GitHub Pages enabled: yes (workflow build_type)
- Custom domain: au-contracts.benrichardson.dev — Cloudflare CNAME created, GitHub Pages CNAME set + cycled for cert
- Deploy workflow: triggered and completed successfully
- Live: https://au-contracts.benrichardson.dev returns HTTP 200
- PR created: https://github.com/ben-gy/au-contracts/pull/1

## Errors & Resolutions
- **OCDS API HTTP 400** on first collect: millisecond precision in ISO timestamps was rejected. Fixed `iso()` to emit seconds + 'Z'.
- **Messy big-firm list**: naive substring matching produced duplicates (DXC/IBM variants) and false positives ("MONEY" etc.). Rewrote with canonical regex patterns aggregated across all rows.
- **TypeScript strict errors** (unused imports/vars, import.meta.env): removed unused imports, added `vite-env.d.ts`.
- **Browser visual verification unavailable**: the Claude Preview MCP port (5200) was held by another session, the Chrome extension was offline, and computer-use is blocked in scheduled runs. Compensated with jsdom render/content tests covering every view + HTTP endpoint checks. Recommend a manual glance at the live site.
- **Repo name conflict**: `gh repo create` failed because an empty `au-contracts` repo already existed on the account; confirmed it was empty (0 commits) and pushed to it directly rather than renaming.
