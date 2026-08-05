# Build Log: Financial Advisers
**Date:** 2026-08-05
**Status:** deployed

## Idea Source

Researched. `IDEAS.md` was empty, so a five-agent parallel sweep (Opus 5) ran over
different discovery angles — federal agencies and regulators, keyless CORS
real-time feeds, state/local open-data portals, global datasets, and
working-backwards-from-search-demand. That produced ~25 candidates, ten of which
went through an adversarial feasibility agent whose brief was to **kill** the
idea: fetch the endpoint, count real rows, check the licence, and verify the
claimed headline. Eight survived; a director agent picked between them.

Winner: **ASIC's Financial Advisers Register** (verifier score 88/100).
Runner-up: the private health insurance product set (87) — held back because
`au-super` already covers a regulated-product comparison in the same *shape*,
whereas the adviser register is structurally new for the fleet.

Two candidate areas were rejected up front against `EXPANSION_IDEAS.md`, which
already records vehicle recalls and CDR retail energy plans as expansions rather
than new sites.

## Site Details
- **Name:** Financial Advisers
- **Repo:** ben-gy/au-advisers
- **Category:** finance / professional regulation
- **Audience:** anyone about to hand money to an adviser (the "check my
  financial adviser" query); compliance officers vetting a hire; the trade
  press; licensee M&A analysts and recruiters
- **Stack:** Vanilla TypeScript + Vite 6, Leaflet 1.9.4 the only shipped
  dependency, every chart hand-rolled SVG
- **Data strategy:** pipeline, **monthly cron** — ASIC republishes the register
  monthly, so the cadence is proportional to the source and is also the fastest
  this factory permits. Day 9, 04:23 UTC, staggered off the hour.

## Why this is genuinely new for the fleet

Grepping `registry.json`, `IDEAS.md` and `EXPANSION_IDEAS.md` for
advis/asic/licensee/planner returns zero hits, and nothing in the fleet is
sourced from ASIC. More importantly the novelty is **structural**: 63 of the 65
existing sites are aggregate statistics — counts per region, per year, per
category. This is the fleet's first **named-person longitudinal register** and
its first dataset that is natively a **directed movement graph** (41,991
transitions). `au-super` is APRA product-level, `au-branches` is bank premises,
`au-insolvency` is AFSA bankruptcy, `au-contracts` is procurement — none of them
contain people, so none could produce a career timeline or an ownership chain.

## Data Sources

| Source | URL | Licence |
|---|---|---|
| ASIC Financial Advisers Register | data.gov.au CKAN pkg `f2b7c2c1-f4ef-4ae9-aba5-45c19e4d3038` (54 MB **tab**-delimited `.csv` with a UTF-8 BOM; 89,060 rows × 76 cols) | CC BY 3.0 AU |
| ABS ASGS 2021 postal areas | `geo.abs.gov.au` ArcGIS, server-side generalised (2,644 polygons, 116,711 vertices) | CC BY 4.0 |
| ABS 2021 Census G01 by POA | `data.api.abs.gov.au` SDMX `C21_G01_POA` | CC BY 4.0 |

**Deliberately not used: ASIC's Banned & Disqualified Persons register.** It has
no adviser number, so the only possible join key is a person's *name*, and one
collision would publish somebody else's banning next to a real named individual.
The Financial Advisers Register already carries ASIC's own disciplinary linkage
(`ADV_DA_TYPE`: 859 actions against 277 people), which is both safer and more
accurate. This narrowing is stated in the plan, the README, the About panel and
the Conduct view itself — it is a correctness decision, not a shortcut.

## Architecture Decisions

- **Vanilla TS.** Ten views with one shared drawer; no component tree deep
  enough to justify React.
- **Sharded detail.** The full per-adviser career detail is 21 MB. Shipping it
  whole would make every visitor download the entire register to read one
  person, so it is split into 83 lazily-fetched shards of ~260 KB. The search
  index that must cover all 42,305 names is shipped **columnar** (one array per
  field) — 2.4 MB, ~635 KB gzipped, about a third of the object-per-row size.
- **Per-view lazy loading.** The Overview costs ~10 KB; the 2.5 MB boundary file
  is only fetched when the map view opens.
- **All thinking in `pipeline/parse.mjs`**, dependency-free and Node-only-import
  free, so the tests exercise the exact code the pipeline runs and CI needs no
  install step for the data job.

## Test Results
- Tests written: **212**
- Tests passed: **212**
- Tests failed: 0

Breakdown: 45 parser, 30 gate, 48 positional layout, 15 overlay-contract, 74
view-integration against the real committed data.

Two failures during development were **real bugs, not test bugs**:

1. The qualification parser split on `/\s+-\s+/`; because `\s+` is greedy it
   consumed the second dash of an empty middle field, gluing it to the
   institution. Most qualifications were silently shipping an empty institution.
   Fixed by splitting on the literal `' - '`, which cannot overlap itself.
2. The overlay scroll-lock was a module-level counter that drifted whenever an
   overlay left by a route the counter did not know about, welding
   `overflow: hidden` onto the body. Replaced with a lock derived from the DOM,
   which cannot drift.

One failure was a genuine **false positive** and the test was corrected rather
than the code: a `\bInfinity\b` check on rendered text fired on the real firm
*Infinity Capital Solutions Pty Ltd*. Non-finite numbers are still asserted
where they matter — in SVG coordinate attributes.

## Build Status
- npm install: pass
- npm test: pass (212/212)
- npm run build: pass
- Local preview: pass
- Cold-start pipeline (`FORCE_DOWNLOAD=1`): pass, 9.4 s, all 8 gates green

## The eight gates

Every gate has a test proving it **fails on the exact fault it claims to catch**.

| Gate | Result |
|---|---|
| row-conservation | 89,060 = 89,060 accepted + 0 rejected |
| status-coherence | Current ⟺ blank end date across 89,060 rows |
| **dating-rule** | 530 phantom advisers; CBA 3575→3238, NAB 3500→3500, Westpac 2414→2414, ANZ 3050→2668, AMP 6518→6339 |
| movement-conservation | 41,991 transitions + 27,204 exits = 69,195 edge weight |
| geography-scope | current 99.8% have a postcode, ceased 35.2% |
| **survivorship** | 12 pre-2015 exits vs 27,192 after; 12 cohorts, earliest 2015 |
| census-anchor | 25,422,756 across 2,643 postcodes = 100.000% of ABS's published 25,422,788 |
| boundary-coverage | 2,644 polygons, 116,711 vertices; 911 mapped, 25 PO-box, 5 malformed |

### The dating rule, and validating the guard where the fault lives

`LICENCE_CONTROLLED_BY` is today's ownership chain stamped onto every historical
row. Counting naively gives 15,949 bank/AMP alumni; intersecting each
appointment's dates against the embedded `[Date Ceased]` markers gives 15,419 —
**530 advisers who were never there**.

The correction is **not uniform**: −337 CBA, −382 ANZ, −179 AMP, and **exactly
zero** for NAB and Westpac, whose cease dates fall after all their advisers'
appointments began. A test written only against NAB or Westpac therefore passes
on an implementation with the date intersection deleted entirely. The gate
asserts a strict gap on the three groups where the fault can appear and exact
equality on the two where it cannot, and there is a test that explicitly
demonstrates the broken build sailing past a NAB/Westpac-only check. This is the
`au-bushfires` lesson made executable.

### The survivorship bias — caught in browser verification, not by tests

The first cohort view charted intakes back to 1995 and reported that **98.8% of
the 1995–2012 intakes were still registered five years later**. Every test
passed. The number was pure artefact.

ASIC's register commenced in 2015. It carries appointments back to 1969 but it
does **not** carry the people who had already left before it existed: of 42,305
advisers, exactly **12** have a last appointment ending before 2015, against
27,192 after. So a "1995 intake" in this file is only the part of that intake
which survived twenty years to still be practising when the register switched
on, and its measured survival is ~100% *by construction*. The give-away was
visual — curves flat at 100% for fifteen years and then a cliff — which is
exactly why the visual pass exists.

Cohorts now start at 2015, keyed on the first **observed appointment** rather
than the self-reported first-advice year (27.7% of advisers report first giving
advice more than three years before their earliest recorded appointment, so the
two are not interchangeable). A `survivorship` gate fails in **both**
directions: if a charted cohort predates the register, and if pre-register exits
ever stop being a rounding error.

The corrected data then refuted the obvious follow-up claim as well. Three-year
retention is **61.1%** for the 2015–17 intakes and **62.2%** for 2020–23 — a
+1.1 point difference, i.e. nothing. Retention did not fall. What collapsed was
the **pipeline**: 5,391 starters in 2018 against 159 in 2020, a 34-fold fall as
the professional-standards regime took effect on 1 January 2019. And the 2019
cohort who entered exactly as the exam and degree requirements landed retained
just **6.2%** at three years. The view now says that, instead of the tidy claim
that was false.

## Per-view UX critique

| View | Question it answers | Hover | Click | Next action |
|---|---|---|---|---|
| Overview | How big is this, and what is the headline? | year-by-year splits | jumps to a view | four routed prompts |
| Bank retreat | How fast, and exactly when, did the banks leave? | per-year owner split | — (toggle drives it) | the naive/dated toggle |
| Movements | Who exchanged people with whom? | edge weight, node size | selects + dims unrelated, opens firm | Clear selection, density control |
| Ownership | How many companies stand behind my adviser's firm? | advisers beneath, cease date | opens licensee | zoom/pan |
| Cohorts | If someone started in year X, how long did they last? | alive/total at each year | — | the fixed-age table |
| Diaspora | My firm died — where did its people go? | flow size and share | opens destination firm | licensee picker |
| Conduct | Do some firms produce disciplined advisers? | firm, numerator and denominator | opens licensee | the ≥200-alumni table |
| Authorisations | What advice can you still get? | share per state | — | the generational split |
| Where | Are there advisers near me? | count, rate, population | opens postcode drawer | best/least served boards |
| Find an adviser | Who is this person? | histogram band | opens the dossier | the dossier's firm links |

Every visually interactive element responds; there are no dead affordances. The
`role="button"` / tab-stop / tooltip properties are asserted by the view tests
across all nine non-map views.

## Deployment
- Repo created: yes — https://github.com/ben-gy/au-advisers
- GitHub Pages enabled: yes, custom domain `au-advisers.benrichardson.dev`
- Cloudflare CNAME created: yes (the zone had 193/200 records, so there was room)
- TLS cert: **approved on the first poll**
- Deploy workflow: **GREEN on first run**
- Data Pipeline workflow: **GREEN on first run** — fetched ASIC + ABS live on a
  GitHub runner and passed all eight gates there, and was **idempotent** (no data
  commit; it embeds the data as-at date, not a run timestamp)
- Directory entry live on main: yes
- Live bundle matches local dist byte-for-byte: yes (`index-nYTwkk24.js`)

## Errors & Resolutions

Four defects were found by **looking at the rendered site**, all of which the
212-test suite passed straight over:

1. **The survivorship bias** (above) — a confident, entirely meaningless
   statistic on the front of a view. Fixed, gated, and the view now explains the
   2015 floor before showing a single number.
2. **The movement network shipped as a structureless dot-cloud** — 595 nodes,
   1,651 links, every label overprinting its neighbours. Rebuilt with a density
   control defaulting to the 20+ adviser backbone (171 firms), hub-only labels
   with white halos, and click-to-trace selection that dims everything a firm is
   not connected to. A click that only opened a panel was wasting the graph.
3. **The ownership icicle was dominated by one aggregated block** — "1,689
   smaller firms" covering 40% of the chart and squashing every real ownership
   chain into 2px slivers. The top level now shows only groups of 40+ advisers,
   with the 1,810 excluded firms and their 7,046 advisers **stated in the
   caption** rather than silently dropped, and depth is measured *after* pruning
   so no empty columns are reserved.
4. **Zoom controls silently never rendered** on the two densest views:
   `attachSvgZoom` was called before the SVG entered the DOM, so
   `svg.parentElement` was null and the controls had nowhere to attach.

Also fixed during verification:

- Streamgraph event labels overprinted each other (2018/2019, 2023/2024); they
  now stagger onto two lines with leader lines.
- The map cut Australia off at 1440×900. A single `requestAnimationFrame` fired
  while the container was still full-bleed, so the fit was computed against the
  wrong box; replaced with a `ResizeObserver` that refits whenever the frame
  actually changes size, and the frame is capped near Australia's own aspect
  ratio so the choropleth is not swimming in ocean.
- `gateBoundaryCoverage` failed on the real file at first — correctly. 30
  adviser postcodes have no ABS boundary: 25 are PO-box/large-volume-receiver
  ranges (2001, 4001, 6909, 7001…) that ABS does not publish as areas, and 5 are
  rows where ASIC's own postcode column contains a state abbreviation, including
  one `MSW` typo. The fix was **not** to raise the threshold but to make the gate
  precise: an unmatched postcode is only acceptable if the census also has no
  population for it, so a populated residential postcode with no polygon still
  fails at any count. All 74 affected advisers are counted and named in the view.
- `attachSvgZoom` threw on `viewBox.baseVal` where no layout engine exists,
  taking the entire view down rather than just its zoom controls; it now falls
  back to parsing the attribute and then to a no-op handle.

## Limitations, stated plainly

- **The map view is not covered by the jsdom view tests** — Leaflet needs a real
  layout engine. It is covered by the live browser verification instead (real
  clicks, tile/vector-layer assertions, and the modal-over-Leaflet test).
- **Authorisations and the map are current-advisers-only**, because ASIC blanks
  those columns on every ceased row. Both views say so.
- **Postcode populations are 2021 Census** while adviser counts are current; the
  vintages differ and the About panel says so.
- Bands in the retreat view count a dual-appointed adviser once per owner while
  the total counts them once, so bands do not sum to the total. Stated in the
  view's own subtitle.
