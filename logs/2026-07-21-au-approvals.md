# Build Log: Building Approvals
**Date:** 2026-07-21
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty. The catalog is deep in ABS/AU government data but had no national **housing supply** site — au-planning covers development applications from one council (City of Casey), not national building approvals. With housing the dominant Australian policy issue and the National Housing Accord (1.2M homes by 2029) actively tracked, national **Building Approvals** was a clear, topical gap. WebSearch confirmed the Accord is running well behind (NHSAC March 2026 quarterly report; ~112k homes behind after 21 months), and the ABS publishes a clean machine-readable Building Approvals series (BA_SA2 SDMX dataflow) with houses-vs-apartments detail down to SA2/SA3 monthly from July 2021.

## Site Details
- **Name:** Building Approvals
- **Repo:** ben-gy/au-approvals
- **Category:** housing
- **Audience:** People following the housing-supply crisis — buyers/renters, journalists, policy watchers, council/industry.
- **Stack:** Vanilla TypeScript + Vite + Vitest
- **Data strategy:** pipeline. Cron **monthly** (`23 5 11 * *`) — ABS releases Building Approvals monthly; monthly is the fastest allowed cadence and matches the source. Staggered to the 11th so it doesn't fire with the rest of the fleet.

## Data Sources
- ABS Building Approvals — `ABS,BA_SA2` SDMX dataflow (data.api.abs.gov.au): new residential dwelling units (New work, Total Sectors) by SA3/state/national, split houses(110)/townhouses(120)/apartments(130)/total(100), plus value (measure 2). Monthly from 2021-07.
- ABS Estimated Resident Population by SA3 — `ERP_ASGS2021`, `ERP.3.TOT.SA3..A` (data.api.abs.gov.au). Used TOT age directly (340 rows), sidestepping the A59="5-9" age-band trap.
- ABS ASGS 2021 SA3 boundaries — geo.abs.gov.au ArcGIS (359 polygons paged, simplified with mapshaper to 882KB).

## Architecture Decisions
- Reused the SA3-grain fleet template (au-insolvency) for the shell, tooltip, glossary, svgZoom, charts and Leaflet z-index/isolation pattern, but wrote all views and the pipeline fresh for the building-approvals data shape (monthly time series + dwelling-type split rather than quarterly counts).
- **SA3 (337 active regions)** chosen as the region grain: legible, stable month-to-month, and matches the proven template. National + state series pulled directly from ABS (STE/AUS) rather than summed, avoiding level-dependent aggregation issues.
- Dependency-free `pipeline/parse.mjs` (its own SDMX CSV parser) so the vitest suite never installs anything; mapshaper is the only pipeline dep and the data-pipeline workflow uses `npm install` (not `npm ci`) per the known CI trap. `parse.mjs` skips blank OBS_VALUE explicitly because `Number('')` is 0, not NaN.
- **Honest framing over a fake trap:** the Accord target counts *completions*, not approvals. The benchmark line is labelled as the completions pace and the glossary explains approvals lead completions and that not every approval is built — the site never claims approvals = target.
- Build-time invariant: SA3 last-12 houses + non-house must equal SA3 last-12 total (110 + 850 = 100 split); the aggregate fails on >0.01% drift.

## Test Results
- Tests written: 33 (across 3 files)
- Tests passed: 33
- Tests failed: 0
- `tests/parse.test.ts` — CSV quoting, SDMX/ERP parsing, state mapping, month axis, alignSeries, rolling/windowSum, buildRegions (synthetic), buildNational, median. One real bug caught and fixed: empty OBS_VALUE was being counted as 0.
- `tests/histogram.test.ts` — position-asserting layout tests (bounds, no pairwise overlap, flush tiling, height ∝ count).
- `tests/svgZoom.test.ts` — zoom/clamp viewBox math (focus stationary, min/max scale).

## Build Status
- npm install: pass
- npm test: pass (33/33)
- npm run build: pass (230KB JS / 71KB gzip, 30KB CSS)
- Local preview: pass

## Deployment
- Repo created: yes (ben-gy/au-approvals)
- GitHub Pages enabled: yes (build_type=workflow, https_enforced)
- DNS: Cloudflare CNAME au-approvals → ben-gy.github.io created; CNAME set + cert cycle triggered
- Workflow triggered: yes (Deploy to GitHub Pages)
- PR created: see registry pr_url

## Verification (local preview, production dist)
- All 9 views render with zero console errors.
- Map: 340 SA3 polygons, skew-aware ramp; drill-down drawer opened **over the map view** paints above Leaflet (z-index 2100, hit-tests inside drawer).
- Real trusted click (Browser pane, genuine pointer) on a Trajectory dot → correct region drawer (Melton - Bacchus Marsh, code 21304).
- No horizontal overflow at 375px on any view, including drawer open (scrollWidth == clientWidth on all 9 + drawer).
- Accord & density rolling-12 charts trimmed to start at the first filled window (no dead left-space).

## Errors & Resolutions
- Empty `OBS_VALUE` counted as 0 (`Number('')===0`) — fixed parse to skip blank observations; caught by a unit test.
- Rolling-12 charts left ~18% dead space on the left (first 11 months are gaps) — added a start offset so the plot fills the width.
- Unused `stateColour` import in density.ts failed `tsc` — removed.
- Preview launch.json needed an absolute `--prefix` because the site lives in the main repo path, not the worktree.
