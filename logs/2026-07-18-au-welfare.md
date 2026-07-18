# Build Log: Welfare Payments
**Date:** 2026-07-18
**Status:** deployed

## Idea Source

Researched. IDEAS.md was empty, so I surveyed uncovered Australian open datasets against a registry
that is already dense in AU government/data explorers. Social security was the one large, fresh,
genuinely uncovered domain: `au-income` is taxable income (ATO), `au-jobs` is the unemployment
*rate* (labour force survey), and neither touches the payment recipient population. The DSS payment
demographics turned out to be published quarterly at SA2 (2,454 areas × 12 quarters), at electoral
division (150 × 6 quarters), and as a 173-month national monthly series — all free, machine-readable
and current to Mar 2026.

Rejected on the same sweep: ACARA school data (the good tables are behind the Data Access Program,
not on data.gov.au) and DSS payments by LGA (same data, coarser geography than SA2 — no reason to
prefer it).

## Site Details
- **Name:** Welfare Payments
- **Repo:** ben-gy/au-welfare
- **Category:** government-transparency
- **Audience:** Council/NGO social planners siting services; journalists writing regional
  disadvantage stories; residents comparing their own suburb.
- **Stack:** Vanilla TypeScript + Vite 6 + Leaflet 1.9 + Vitest
- **Data strategy:** pipeline, `17 6 9 2,5,8,11 *` — quarterly, one month after each DSS quarterly
  release, staggered to day 9 at an off-hour. The source does not change between releases, so
  anything faster would be pure waste.

## Data Sources
- DSS Payments by 2021 SA2 — https://data.gov.au/data/dataset/dss-payments-by-statistical-area-2
- DSS Payments by 2024 CED — https://data.gov.au/data/dataset/dss-payments-by-commonwealth-electoral-division
- DSS Income Support Recipients Monthly Time Series — https://data.gov.au/data/dataset/dss-income-support-recipients-monthly-time-series
- ABS ERP by SA2 and age, `ERP_ASGS2021` — https://data.api.abs.gov.au
- ABS ASGS 2021 SA2_GEN boundaries — https://geo.abs.gov.au
- Digital Atlas of Australia federal electorates (Mar 2025) — reused from `au-mp-expenses`

## The design decision the site is built on

Raw recipient counts are unreadable, which is why this dataset is largely ignored. The value is in
the join to ABS population **by age band**, which makes three derived measures possible:

1. **Working-age income support rate** — income support excluding the Age Pension, over the 15–64
   population. 15.7% nationally. This is the disadvantage measure.
2. **Age Pension take-up** — Age Pension recipients over the 65+ population. 55.9% nationally;
   12.2% in Bellevue Hill, 13.3% in Toorak. Because the pension is means-tested, low take-up is a
   wealth signal.
3. **DSP : JobSeeker ratio** — 1.79 in Mount Hutton – Windale (Newcastle), 0.21 in West Arnhem.
   Structurally different labour markets that look identical in a headline rate.

The headline-vs-working-age divergence is real and large, which is what justified making the scatter
the signature view rather than another ranking: Tea Gardens – Hawks Nest is 40.0% headline / 24.9%
working-age; Elizabeth (SA) is 50.2% / 61.5%. No one-dimensional view can separate those.

## Architecture Decisions

- **Vanilla TS, not React.** Nine views with no shared component tree and no routing — view
  switching plus one drawer. React would have added weight for nothing.
- **Column-oriented `regions.json`.** 2,454 × 40 as arrays-of-arrays with a declared column order,
  expanded to objects once at boot: 577 KB instead of ~1.5 MB of repeated keys.
- **Suppress, don't clamp.** Industrial-estate and airport SA2s (1–4 residents) produced Age Pension
  take-up rates up to 500% against DSS's round-to-5. Rates below a denominator floor
  (500 total / 400 working-age / 200 senior) are emitted as `null` and excluded from rankings, map
  and scatter, rather than pinned to 100% where they would top every leaderboard.
- **Electorates carry counts only.** The ABS publishes no population for 2024 electoral boundaries.
  Substituting AEC enrolment or 2021-CED population would have produced authoritative-looking wrong
  rates, so the view uses counts and composition, and says why.
- **Never sum supplements into the total.** Only the 11 mutually exclusive income support payments
  form a headcount; Rent Assistance, FTB, Carer Allowance and the four concession cards are held
  *alongside* a payment and would double-count.
- **Boundaries:** ABS SA2_GEN (layer 1 of the SA2 MapServer — the generalised geometry) simplified
  with mapshaper to 1.5% retention → 1.94 MB / ~400 KB gzipped, matching the shipped precedent in
  `au-income` (1.2% / 1.6 MB for 2,266 comparable polygons). Verified visually on production: real
  coastlines, Tasmania and the island groups all intact, nothing blocky.
- **Patterns reused, not re-rolled:** `tooltip.ts`, `svgZoom.ts`, `leafletMap.ts` (adapted),
  `tests/layout.test.ts`.

## Test Results
- Tests written: 119 (4 files: format, analysis, data/pipeline parsers, positional layout)
- Tests passed: 119
- Tests failed: 0

Three initial failures were all wrong *expectations*, not wrong code, and were corrected:
`formatCompact(2795000)` expected `2.80m` where binary floating point yields `2.79m`; a sparkline
assertion expected `M0,` against the emitted `M0.00,`; and a CSV fixture used a one-column header
row that the parser correctly discards as a trailing footnote.

## Build Status
- npm install: pass
- npm test: pass (119/119)
- npm run build: pass
- Local preview: pass
- Production verification: pass

## Per-view UX critique (recorded per the mandatory check)

| View | Question it answers | Hover | Click | Next action |
|---|---|---|---|---|
| Map | Where are the concentrations, and what does my area look like? | Full stat block per SA2 | Opens the area profile | Profile cross-links to rank and trend |
| Pension Illusion | Is this area's welfare receipt about age, or about disadvantage? | Both rates + take-up + population | Opens the profile | Ranked side panels list both extremes |
| Rankings | Who is highest/lowest, and how far from typical? | Value vs median vs population | Opens the profile | Metric/state/floor switches |
| Explorer | What are the numbers for the place I came here for? | Sparkline shows every quarter | Opens the profile | Sort or search onward |
| Electorates | How does this map onto federal politics? | Counts per division | Selects and isolates the division | Composition strip below updates |
| Payment Mix | What *kind* of support is this? | Count + share + payment definition | Opens the profile | Sort by any family |
| Trends | How did we get here? | Every payment's value in that month | Legend toggles series | Indexed mode for small payments |
| Distribution | Is my suburb normal? | Bin range + area count | Filters the Explorer to that bin | Lands in Explorer with a clearable chip |
| Insights | What should I have noticed? | — (text cards) | Opens the area profile | — |

Every visually-interactive element was walked; no dead clicks remain.

## Errors & Resolutions

1. **ABS SDMX returned 406.** The endpoint content-negotiates and rejects a request without an
   explicit `Accept`. Added `Accept: text/csv` to that fetch.
2. **ABS SA2 boundary 404.** `SA2_GEN` is not its own service — it is layer 1 of the `SA2`
   MapServer. Corrected the URL.
3. **Boundary file 11.4 MB.** Retuned mapshaper from 12% to 1.5% retention at precision 0.001 →
   1.94 MB / 400 KB gzipped, checked against the au-income precedent and verified visually.
4. **Age Pension take-up of up to 500%.** Industrial/airport SA2s with 1–4 residents aged 65+ against
   DSS's round-to-5. Introduced denominator floors with `null` suppression (161 areas affected for
   take-up, 123 for working-age).
5. **"National figure" tile rendered an em dash.** Metrics are keyed on the Region field
   (`rateWorking`) while `summary.nationalRates` uses short names (`working`), so the lookup silently
   missed. Added an explicit `natKey` mapping plus a regression test that asserts every rate metric
   resolves to a real national key.
6. **Every ranked bar rendered empty.** `.rank-fill` is a `<span>` and the CSS never set
   `display:block`, so width/height were ignored on an inline box — invisible bars in three views.
7. **Tooltips never hid.** The copied tooltip util toggles a `.visible` class my stylesheet never
   defined, so the tip was created visible and stayed. Added the hidden/visible rules, plus an
   exported `hideTooltip()` called on view switch (swapping views mid-hover destroys the anchor
   before `mouseout` can fire, stranding the tip).
8. **Trends annotation labels collided.** "COVID-19: JobSeeker doubles" and "Supplement withdrawn"
   shared a baseline. Staggered onto alternating rows with edge-flipping, and added greedy
   de-collision for the end-of-line series labels.
9. **Deploy failed on CI (first push).** `tests/data.test.ts` imported `pipeline/aggregate.mjs`,
   which imports mapshaper — installed only under `pipeline/`, so the root `npm ci` could not resolve
   it. Split every pure parser into `pipeline/parse.mjs` (node builtins only) and pointed the tests
   there; `aggregate.mjs` is now orchestration and IO. Re-ran the full pipeline afterwards and
   confirmed byte-identical output.
10. **Data pipeline failed on `npm ci`.** mapshaper pulls a wildcard `@types/node`, so even a freshly
    generated lockfile fails `EUSAGE`. That step now uses `npm install` and the pipeline lockfile is
    no longer tracked. Both workflows are green.

## Deployment
- Repo created: yes — https://github.com/ben-gy/au-welfare
- GitHub Pages enabled: yes
- Cloudflare DNS: yes (CNAME au-welfare → ben-gy.github.io)
- TLS certificate: approved, `https_enforced=true`
- Deploy workflow: success
- Data pipeline workflow: success
- PR created: https://github.com/ben-gy/au-welfare/pull/1

## Production verification

Live bundle hash matched the local build. All nine views walked on
`https://au-welfare.benrichardson.dev` with zero console errors. Real pointer-event clicks confirmed:
map polygon → profile (Bourke – Brewarrina, 48.4% working-age, +32.7pp), matrix cell → profile
(Whyalla), electorate row → isolates Spence, histogram bin → Explorer filtered to 258 areas with a
clearable chip. Scatter zoom-in then drag panned the viewBox without firing a click. The About modal
was opened *from the map view* and screenshotted sitting fully above Leaflet. At 375px every view
reported `scrollWidth === clientWidth`, including with the drawer and modal open.

Note on tooling: this browser pane renders screenshots at a different scale from `window.innerWidth`,
and at one point reported `innerWidth: 0` after a resize — which briefly looked like a zero-width
layout bug. Layout was therefore verified with explicit `getBoundingClientRect()` assertions rather
than by eye, and screenshots used for content and colour.
