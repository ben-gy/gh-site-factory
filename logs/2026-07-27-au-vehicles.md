# Build Log: Vehicle Fleet
**Date:** 2026-07-27
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty. Rejected ACCC Scamwatch (rich data but locked behind a Power BI
dashboard with no clean bulk download — too fragile for a pipeline). Landed on **BITRE Road Vehicles
Australia** (the successor to the discontinued ABS Motor Vehicle Census) — a clean, stable, annual
data.gov.au dataset with postcode × fuel × year-of-manufacture × make/model granularity. Genuinely new
domain (no vehicle-registration site in the catalog; au-flights is aviation, au-crashes is road safety).

## Site Details
- **Name:** Vehicle Fleet
- **Repo:** ben-gy/au-vehicles
- **Category:** transport (index category: data-explorers)
- **Audience:** EV-curious buyers, journalists, policy people, transport nerds
- **Stack:** Vanilla TypeScript + Vite + Vitest; Leaflet map; hand-rolled SVG charts
- **Data strategy:** pipeline, **annual** cron (`41 5 11 8 *`) — BITRE publishes a January registry
  snapshot once a year, released mid-year. The slowest proportional cadence.

## Data Sources
- BITRE Road Vehicles Australia, January 2021–2025 (data.gov.au dataset `road-vehicles-australia-january-2025` + sibling years), resolved via the CKAN `package_show` API by filename pattern.
- ABS ASGS 2021 Postal Area digital boundaries (abs.gov.au) → mapshaper-simplified GeoJSON.
- matthewproctor/australianpostcodes (postcode → locality/lat/lng).

## The story
- Battery-electric = **1.55%** of the passenger fleet; diesel = **14.6%** → **9.4× more diesel than electric**.
- Median build year **2015** (~10 yr old fleet). Whole fleet **5.9%** electrified; cars built in 2024 **35.5%** electrified, **8.6%** fully electric — the stock-vs-flow gap the New-Car Effect slider teaches.
- Diesel rose every year 2021→2025; light-commercial fleet 77% diesel, trucks 97–99%, motorcycles 99% petrol.
- ACT leads EV share (3.5% BEV / 9.3% electrified); SA/TAS oldest & least electric.
- Toyota 3.25M cars; Tesla dominates EVs (Model 3 63k, Model Y 59k), BYD/MG rising.

## Architecture Decisions
- Vanilla TS + Vite (fleet default; closest analog au-solar reused for the app shell, POA choropleth,
  drawer, tooltip/glossary/about patterns).
- Reused the au-solar POA boundary fetch+mapshaper approach; `npx -y mapshaper` (no pipeline `npm ci`,
  avoiding the CI trap).
- Fuel colours defined once in `src/fuels.ts` and referenced everywhere (donut, area, matrix, map, bars,
  mini-bars) so a colour means the same fuel across all views.
- Reconciliation is tolerant (perturbed data) — see traps below.

## Data traps handled
1. `'Total'` pseudo-state row excluded (zeroed in the detailed extract but would otherwise rank as a state).
2. Year-of-manufacture `'-'` / `'Not stated'` sentinels dropped from age maths.
3. Six motive-power labels canonicalised to five fuel classes (Dual fuel/Other/`-` → Other).
4. **Perturbation + <3 suppression:** interior sums never equal published totals → hard gate is tolerant:
   state-file vs postcode-file national passenger totals must agree within 1.5% (matched to **0.02%**),
   and the national total must be in range, or the build fails.
5. Boundary `POA_CODE21` → `pc` rename in aggregate (caught in verification: every polygon was painting
   "no data" grey before the fix).

## Test Results
- Tests written: 37 (parse/canonicalisation/reconciliation traps, squarified-treemap positional layout,
  analysis/ranking/histogram/insights, formatters, svgZoom pure math).
- Tests passed: 37 / failed: 0. (Two initial failures were wrong test *expectations* — data below
  `MIN_VEHICLES` and a wrong histogram bin index — fixed; the code was correct.)

## Build Status
- npm install: pass
- npm test: pass (37/37)
- npm run build: pass (68 KB gzip JS)
- Local preview: pass (HTTP 200; all data + SEO assets present in dist)

## Verification (local production build, real trusted clicks)
- All 9 views render, no NaN, **zero console errors**.
- New-Car Effect slider (real click) → MY2024 shows 35.5% electrified (vs 5.9% whole fleet).
- Real map-polygon click → per-postcode drawer (6440, 41.7% diesel) painting **above Leaflet** (z 2100).
- About modal opened **from the map view** paints above Leaflet (z 2100).
- Squarified make treemap + type×fuel matrix + all charts render correctly.
- Explorer row click opens the drawer; histogram bars render.
- **Zero page-level horizontal overflow at a true 375px viewport** across all 9 views + an open drawer.
- Fixed during verification: (a) the POA_CODE21→pc join bug (grey map); (b) green ramp break points
  re-set from the real fuel-share quantiles (a near-white low end washed the map out on skewed data).

## Deployment
- Repo created: yes (ben-gy/au-vehicles)
- GitHub Pages enabled: yes (workflow build type)
- Cloudflare DNS CNAME: created (au-vehicles → ben-gy.github.io)
- Deploy workflow: green on first run
- PR created: https://github.com/ben-gy/au-vehicles/pull/1
- TLS cert: `https_enforced` still issuing at hand-over (HTTP served, deploy green) — cycle applied; re-check.

## Errors & Resolutions
- Scamwatch idea rejected early (Power BI, no bulk download).
- `tsc` failed with tests in `include` (untyped `.mjs` import) → reverted tsconfig `include` to `["src"]`
  (vitest transpiles tests independently). 
- Choropleth painted grey (POA_CODE21 vs `pc`) → aggregate renames the field. Fixed & re-verified.
