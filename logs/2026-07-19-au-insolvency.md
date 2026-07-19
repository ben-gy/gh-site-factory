# Build Log: Personal Insolvency
**Date:** 2026-07-19
**Status:** deployed (TLS cert pending at hand-over — registry status `verify_production_pending`)

## Idea Source

Researched. `IDEAS.md` was empty, so I searched for current AU open-data candidates and shortlisted
three: higher education / graduate outcomes, AFSA personal insolvency, and NGER greenhouse emissions.

- **Higher education** was rejected on data risk: the Department of Education publishes pivot-shaped
  XLSX and QILT publishes PDF report tables, so the wrangling cost was high and uncertain.
- **NGER emissions** was too close to the existing `au-pollution` (National Pollutant Inventory) to
  count as a distinct product.
- **AFSA personal insolvency** won: clean CSVs on a stable overwrite URL, 75 quarters of history,
  SA3 geography (boundary + ERP fetch patterns already proven in `au-immunisation` / `au-welfare`),
  and no existing site in the fleet measuring financial collapse. Distinct from `au-jobs`
  (unemployment), `au-income` (earnings) and `au-welfare` (benefits) in outcome, geography level and
  data shape.

## Site Details
- **Name:** Personal Insolvency
- **Repo:** ben-gy/au-insolvency
- **Category:** economy
- **Audience:** financial counsellors and community legal centres, cost-of-living journalists, policy
  analysts, and people checking their own region
- **Stack:** Vite + vanilla TypeScript + Leaflet, Vitest
- **Data strategy:** pipeline. AFSA publishes **quarterly**, so the cron is quarterly
  (`23 4 9 2,5,8,11 *`) — one month into each new quarter, staggered to day 9 / 04:23 UTC.

## Data Sources
- AFSA regional quarterly time series — https://www.afsa.gov.au/sites/default/files/2024-08/regional_quarterly_time_series.csv
- AFSA quarterly personal insolvencies — https://www.afsa.gov.au/sites/default/files/2024-08/quarterly_personal_insolvencies.csv
- ABS ERP by SA3 and age (SDMX) — https://data.api.abs.gov.au/rest/data/ERP_ASGS2021
- ABS ASGS 2021 SA3 boundaries — https://geo.abs.gov.au/arcgis/rest/services/ASGS2021/SA3/MapServer

Both AFSA files sit on a stable overwrite URL — the `2024-08` path still serves the Mar-2026
quarter — so the pipeline does not need to rediscover them each run.

## Architecture Decisions

**Vanilla TS over React:** nine views sharing one dataset and a hash router; no component tree deep
enough to justify React.

**The signature view changed during the build.** The plan's headline was a business-vs-consumer
scatter, on the assumption that AFSA's `In a business or company` flag was usable per region. Measuring
it first showed median split coverage of only 54.5% and just 75 of 340 regions fully published — so
that axis would have been a lower bound dressed up as a proportion. The split moved to state level
(where cells are large enough to be reliable) with the withheld share disclosed, and the SA3 signature
became **rate against four-year change**, which the totals fully support and which answers a question a
league table cannot: is this region always bad, or getting worse?

**Rolling four-quarter windows** everywhere. Single quarters for one SA3 are small and seasonal, and
ranking on one noisy quarter reshuffles the leaderboard every release.

**Denominator is 15+, not total population**, labelled explicitly. Bankruptcy requires 18, but the ABS
publishes no 18+ band for small areas; 15+ shifts every region near-proportionally so the ranking is
unaffected.

## Data traps found and handled

Four in the AFSA data, one in the ABS data. Each is documented in `pipeline/parse.mjs` and covered by
tests.

1. **Aggregate rows masquerade as regions.** 364 `ASGS Code` values, only 340 SA3s: 16 are GCCSA
   aggregates (`1GSYD`, `1RNSW`) that *contain* those SA3s, 8 are `"NSW - not in ASGS"` uncoded-address
   buckets. Ranking them together counts the country twice — 463,770 people sat in the aggregate rows.
2. **Two levels of subtotal in the type column.** `Total bankruptcies` = debtor's petitions +
   sequestration orders, and `Total personal insolvencies` = everything. My first aggregation summed the
   column and produced 14,491 for Sep-2007 against a true ~8,089 — every bankruptcy counted twice. The
   pipeline now asserts its leaf sum reproduces AFSA's own published subtotal exactly (3,161 for
   Mar-2026) and fails the build on any drift.
3. **`Data not available` is a string sentinel in a numeric column**, 18,908 times. `Number()` yields
   NaN, but `parseInt` and `Number(x) || 0` both yield a convincing **0**.
4. **Suppression is complementary, and a withheld cell is not a zero.** Where either side of the
   business/consumer split would identify someone, both sides are withheld so neither is recoverable by
   subtraction (measured: 8,259 cells suppress both, 50 suppress one). **Region totals are withheld
   too — 8.9% of cells.** Because genuine zeros *are* published as `0`, a withheld quarter is 1–2 cases,
   so twelve-month totals for affected regions are midpoint estimates carrying an explicit range.
5. **ABS `A59` means ages 5–9**, not 55–59, and sorts innocently between `A55` and `A60`. Summing the
   adult bands by name puts primary schoolers in the per-capita denominator. Verified against `CL_AGE`.

Trap 4 was the significant one: it was caught only because the Insights view claimed **"21 regions
recorded no insolvencies at all"** naming Hawkesbury and Manly — populous Sydney SA3s where that is
impossible. Three of their last four quarters were withheld and were summing to a fabricated zero.
After the fix, true zeros dropped from 21 to 5 (all genuinely tiny: Lord Howe Island, a catchment
reserve with 3 residents), and the "deteriorating fastest" finding changed from a noise artifact
(Upper Murray, 31 cases) to **Melton – Bacchus Marsh** (161 cases, a real Melbourne growth corridor).

## Per-view UX critique

| View | Question it answers | Hover | Click | Next action |
|---|---|---|---|---|
| Rankings | Which regions are worst? | — | row → drill-down | switch metric, filter state |
| Map | Where is it, spatially? | polygon tooltip with 4 figures | polygon → drill-down | switch measure |
| Trajectory | Bad, or getting worse? | dot tooltip w/ rate, count, change, quadrant | dot → drill-down | toggle quadrants via legend |
| Explorer | How does *my* region compare? | `*` tip gives the estimate range | row → drill-down | sort any column, search |
| Timeline | What happened over 19 years? | per-quarter split | — | reads into Kinds |
| Kinds | Which instrument, and who started it? | bars + matrix cells | — | reads into Business vs consumer |
| Business vs consumer | Business failure or household debt? | bars, table | row → drill-down | — |
| Distribution | How unequal is it? | bar tip w/ count and band | bar → lists regions in band | "Open in Explorer" |
| Insights | What should I already know? | — | "Open this region" | jumps to drill-down |

No dead clicks: every `cursor:pointer` affordance is wired. Legend swatches in Trajectory filter
quadrants; histogram bars filter and cross-link.

## Test Results
- Tests written: **107** (parse 48, layout 32, analysis 27)
- Tests passed: **107** — Tests failed: 0

Three genuine bugs were caught by tests rather than by eye:
- `ticks(97)` stopped at 75, so the top gridline never reached the tallest bar
- `shapeNational` derived quarters from only one of two maps, dropping split-only quarters
- the fixture update for trap 4 confirmed the zero-insolvency guard actually fires

## Build Status
- npm install: pass
- npm test: pass (107)
- npm run build: pass (`tsc` clean, 237 KB JS / 72.6 KB gzip)
- Local preview of production dist: pass

## Deployment
- Repo created: yes — ben-gy/au-insolvency
- GitHub Pages enabled: yes, Deploy workflow succeeded
- Cloudflare DNS: CNAME created, resolving to GitHub Pages IPs
- PR created: https://github.com/ben-gy/au-insolvency/pull/1
- Data pipeline workflow: triggered on push and running

## Verification

Against the **production build** (`vite preview` of `dist/`; deployed bundle hash `index-CSh68CGt.js`
confirmed byte-identical to the live one):

- All 9 views render; **zero console errors**
- **Real pointer clicks** (never synthetic `.click()`): map polygon → `#v=map&r=31502` "Outback - North";
  scatter dot → `#v=trajectory&r=21305` "Wyndham"
- Drag on the scatter pans **without** firing the click
- About modal opened **from the map view** paints above Leaflet's panes (screenshot-confirmed)
- No horizontal overflow at 375px across all 9 views, and with the drill-down open
- Live site verified over `http://`: all data, SEO and IndexNow assets return 200; `meta.json` reports
  340 regions, 218 exact, latest Mar 2026

## Errors & Resolutions

- **Business-flag partition guard fired on first run** (11.27% drift). Investigating showed it was not
  a mismatch but suppression — this is what led to traps 3 and 4 and reshaped the site's signature view.
- **Sticky table headers parked mid-table.** Two causes: `--header-h` (54px) was the title-bar height,
  not the full header including nav (~99px); and `overflow-x: auto` on the table wrapper makes it the
  *scrollport* for `position: sticky`, so the offset pushed the header down **inside** the panel.
  Fixed with a measured `--sticky-top` and a `.table-scroll` class that is not a scroll container on
  desktop.
- **Self-inflicted feedback loop:** writing the measured header height back into `--header-h` also fed
  `.header-inner`'s `min-height`, growing the header 54 → 99 → 144px on each observation. Split into a
  separate `--sticky-top`.
- **Annotation labels overprinted** ("DA refCOVMD") on the narrow drill-down chart. Staggered down the
  plot with edge-flipping.
- **TLS cert still issuing at hand-over** after ~15 minutes of polling and two CNAME cycles. Normal for
  a freshly pointed domain. Site is live and verified over `http://`; registry status set to
  `verify_production_pending` with a note, and called out in the PR and REVIEW.md.
- Harness note (not a site bug): the browser tool's click coordinates are scaled 1.6× relative to CSS
  pixels, which made early map clicks land off-screen and look like dead clicks.
