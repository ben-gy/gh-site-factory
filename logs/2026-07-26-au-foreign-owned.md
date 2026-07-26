# Build Log: Foreign Ownership
**Date:** 2026-07-26
**Status:** deployed

## Idea Source
Researched (IDEAS.md was empty). Searched for newly-usable government open datasets; found the ATO **Register of Foreign Ownership of Australian Assets** on data.gov.au — six machine-readable asset-class workbooks (agricultural land, water, residential, commercial, business, mining), all published together as of the 30 June 2025 report. Verified: genuinely new to the fleet (no foreign-ownership/FIRB site exists among the 55 registry entries or the gh repo list), highly searchable ("who owns Australia", "Chinese ownership of Australian farmland"), politically salient, and rich (country × state × asset-class × time). Rejected corporate-insolvency (ASIC) as a runner-up because it risked reading as an expansion of au-insolvency and had a thinner geographic/country dimension.

## Site Details
- **Name:** Foreign Ownership
- **Repo:** ben-gy/au-foreign-owned
- **Category:** government-transparency (index: data-explorers)
- **Audience:** journalists, rural/regional communities, policy researchers, property-market watchers, politically-engaged citizens
- **Stack:** Vanilla TypeScript + Vite + Vitest; Leaflet map
- **Data strategy:** pipeline, **annual** cron (`41 4 8 12 *` — the ATO reports once a year as at 30 June; staggered day 8 Dec to avoid fleet collisions). Dependency-free (Node built-ins only), so no `npm ci` in the pipeline workflow.

## Data Sources
- ATO Register of Foreign Ownership of Australian Assets — six xlsx (data.gov.au dataset `register-of-foreign-ownership-of-australian-assets`, uuid `00e08863-57e7-431a-b5b7-202f8ddd3112`)
- ABS ASGS 2021 state & territory boundaries (`patterns/geo/au-states.geojson`)

## Architecture Decisions
- **Vanilla TS** (fleet default) — single-page tabbed explorer, one Leaflet map, hand-rolled SVG for every other chart.
- **Unifying spine = country of control** across six asset classes for the latest year (2025), because the classes use incompatible units (ha, GL, $, counts) and can't be summed. The `matrix.json` payload drives the Atlas, leaderboard, explorer and per-country drawer.
- **Signature view = a country × asset-class rank matrix** (the "it flips by asset" inversion), backed by ha-trend lines for the China-retreat story.
- Reconciliation gate is the ag "Level" total == "Foreign/Australian share" total (exact) — two independently-published sheets, so agreement proves the serial-date + mislabelled-unit parsing is correct.

## Test Results
- Tests written: 29 (16 parser/helper + reconciliation, 13 treemap positional layout)
- Tests passed: 29
- Tests failed: 0

## Build Status
- npm install: pass
- npm test: pass (29)
- npm run build: pass (clean, no warnings; 214KB JS / 66KB gzip)
- Local preview (built minified dist): pass — all 8 views, real clicks, no console errors, no 375px overflow

## Deployment
- Repo created: yes (ben-gy/au-foreign-owned)
- GitHub Pages enabled: yes (workflow build)
- Cloudflare DNS: CNAME au-foreign-owned -> ben-gy.github.io created OK
- Deploy workflow: green
- Data Pipeline workflow: green (after URL fix, see below)
- PR created: https://github.com/ben-gy/au-foreign-owned/pull/1
- TLS cert (https_enforced): still issuing at hand-over; HTTP 200 live, byte-identical bundle. Should flip on its own.

## Errors & Resolutions
- **CI pipeline failed (data.gov.au HTTP 302):** the download URL concatenated `.../resource` + uuid with no slash (`resourceb5d6...`), which 302-redirected. It passed locally only because the local run used a cached-file path (`LOCAL_XLSX_DIR`). Fixed BASE to end with `/resource/`; verified the corrected URL returns 200 (60KB) against the live host; pipeline re-ran green.
- **Sticky thead parked mid-table** in the Atlas matrix, Explorer, and drawer/modal tables (`overflow-x:auto` / drawer scroll containers capture `position:sticky`). Fixed: matrix + drawer/modal theads made static; `.table-scroll` reverted to `overflow-x:visible` on desktop, `auto` + `top:0` only ≤900px.
- **Window scroll not reset on view change** (the app scrolls the window, not view-root) — added `window.scrollTo(0,0)` in render.
- **About modal not dismissed on navigation** and a **stale hover-tooltip lingering** across views — both fixed in render (dismiss modal, `hideTip()`).
- **TS assigned-type narrowing** on the Atlas sort variable — typed as `string`, cast when indexing.

## Data traps handled (in the dependency-free pipeline)
Excel-serial dates; column-header units that LIE (raw ha/ML/$ under "'000 hectares"/"gigalitres"/"$ million" labels) and switch mid-column (ag Table 1 first year; water country table's last two columns) — normalised by magnitude and cross-checked against the reliable published % column; `<=5` small-cell suppression → null (never zero); country-name case/punctuation/typo variants canonicalised to one ISO-coded entity across the six workbooks; NSW/ACT combined in ag/water/mining (ACT inherits the map value); business interests have no state breakdown.
