# Build Log: Mortality Atlas
**Date:** 2026-07-22
**Status:** deployed

## Idea Source

Researched. `IDEAS.md` was empty (no lines starting with `-`), so I went looking for a
domain the fleet does not already cover. The gap was obvious once listed: 52 registry
entries covering hospitals, vaccination, medicines, aged care, childcare, welfare and
insolvency — and nothing on **mortality or causes of death**, the most fundamental health
outcome there is.

Two candidates were assessed against the criteria:

1. **AIHW MORT books** (chosen) — regional mortality with age-standardised rates,
   potentially avoidable deaths, years of life lost and leading causes.
2. **ABS Corrective Services / Prisoners in Australia** — imprisonment rates by state.
   Rejected as the weaker fit: it overlaps `au-crime`'s domain, is state-level only (no
   fine geography to map), and the quarterly cube is much thinner than MORT.

A third idea, **NGER / Safeguard Mechanism greenhouse emissions by facility**, was
deliberately NOT built: it is the same data *shape* as the existing `au-pollution`
(facility-level emissions, map, treemap, rankings) and a user would reasonably say "that's
the same site with different numbers". Per the expansion rule it was logged in
`EXPANSION_IDEAS.md` against `au-pollution`, with both Clean Energy Regulator source URLs
and the specific views that would make it additive rather than duplicative.

MORT was verified for feasibility before committing: `data.gov.au` package `mort-books`
exposes both tables as stable-resource-id CSVs (2.1 MB and 8.5 MB), no auth, browser-UA
fetch, and the SA3 codes join cleanly to ABS ASGS 2021 boundaries (`SA310102` → `10102`).
I mined the raw data in Python *before* designing anything, so the plan was built around
findings that were already confirmed to exist rather than hoped for.

## Site Details
- **Name:** Mortality Atlas
- **Repo:** ben-gy/au-mortality
- **Category:** health
- **Audience:** PHN/LHD analysts and health planners; journalists covering health
  inequality; residents searching for death rates in their own area
- **Stack:** Vanilla TypeScript + Vite 6 + Vitest, Leaflet 1.9, hand-rolled SVG
- **Data strategy:** pipeline — **yearly cron** (`43 4 9 3 *`). The AIHW republishes MORT
  once a year, so any faster cadence would re-download identical numbers. Monthly would be
  the fleet maximum anyway; annual is what proportionality demands here.

## Data Sources

- AIHW MORT books Table 1 — summary measures by 8 geography levels × 5 years × 3 sexes
  (15,855 rows): https://data.gov.au/data/dataset/mort-books
- AIHW MORT books Table 2 — top-20 leading causes per geography × sex, pooled 2020–2024
  (65,868 rows, 128 distinct causes)
- ABS ASGS 2021 SA3 boundaries:
  https://geo.abs.gov.au/arcgis/rest/services/ASGS2021/SA3/MapServer/1

Both AIHW tables derive from the National Mortality Database (state/territory death
registrations, cause coded by the ABS). Published CC BY 4.0.

## Architecture Decisions

**Vanilla TS over React** — one page, nine tabbed views, no routing tree or deep component
state. A framework would add bundle weight and nothing else.

**No charting library** — every mark needs a `data-tip` and a click target, which is easier
to guarantee by building SVG nodes directly than by fighting a library's render pipeline.
`squarify.ts`, `svgZoom.ts`, `tooltip.ts` and `feedback.ts` were **copied from
`patterns/`** rather than re-rolled.

**A slope chart, not a scatter, for the signature view.** A scatter of crude against
age-standardised rate shows the two correlate — true and useless. The *point* of this
dataset is that individual regions swap ends of the ranking, and only connected rank lines
make a crossing visible. The crossing IS the argument.

**Ratio-to-national colour cuts, not quantiles.** Quantile buckets always fill every class,
so they manufacture visible difference where regions are nearly identical, and the legend
means something different on every measure. Cutting on the ratio to the national figure
lets one legend read "35% above national" on all six map measures. The data supports it —
about a 3× spread end to end, not the orders of magnitude that would demand a log scale.

**Suppression is excluded from rankings, never sorted to an end.** A withheld rate is
unknown; parking unknowns at the bottom would present the most remote communities in
Australia as the country's healthiest.

**Deliberately not built:** a force-directed network (nothing connects to anything here —
it would be decoration) and a Sankey (no value flows between categories; deaths do not move
from one region to another).

## Per-view UX critique

| View | Question it answers | Hover | Click | Next action |
|---|---|---|---|---|
| Overview | How many die, how many needn't have, who carries it? | tiles + every gradient bar | cause bar → Causes view | into the two gradients |
| Age Illusion | Is this region actually unhealthy, or just old? | full numbers per line | line/row → region drawer | into Rankings/Explorer |
| Map | Is this remote, outer-suburban, or everywhere? | Leaflet tooltip w/ ratio + population | polygon → region drawer | measure switcher |
| Rankings | Which regions carry the most burden? | bar tooltip w/ rank + population | bar → drawer; hist bar → filtered Explorer | histogram click-through |
| Causes | What kills Australians, and where worst? | treemap + bar tooltips | cause → "where it hits hardest"; region → drawer | back to all causes |
| Gradients | How much tracks remoteness and disadvantage? | every dot, all three sexes | state dot → drawer | measure switcher |
| Matrix | Do remote Australians die of *different* things? | every cell w/ ratio + deaths | — (dense read-only grid) | group switcher |
| Explorer | What are my region's numbers? | column defs + suppressed cells | row → drawer | sort/search/filter chip |
| Insights | What should I already know? | — | card → the view that shows it | jumps into context |

Every visually-interactive element was walked; no dead clicks remained. The Matrix is
read-only by design and carries no click affordance (no `cursor: pointer`).

## Test Results
- Tests written: **111** (3 files)
- Tests passed: 111
- Tests failed: 0

Two failures during development were both bugs in my own *fixtures*, not the code, and both
were fixed by correcting the test: the CSV fixture builder joined fields with commas without
quoting (so a `"1,000"` value split into two columns), and one expectation assumed
`rankRegions` sorts median-age best-first when rank 1 correctly means "most burden" for
every measure.

Coverage highlights — each test targets a trap that produced a wrong number:
- `parse.test.ts` (41): RFC-4180 quoting, CP1252 decoding (0x96 → en dash), suppression
  sentinels never becoming 0, last-parenthetical ICD splitting, blank-rank subtotal
  detection, ASGS state derivation, and the cross-table reconciliation assertion itself.
- `layout.test.ts` (29): positional treemap assertions (in-bounds, **no pairwise overlap**,
  no NaN, area conservation, degenerate input), zoom clamping, histogram bin contiguity.
- `analysis.test.ts` (41): suppression handling end-to-end, rank-flip denominators, sort
  direction, label de-collision, and every auto-insight's computed ratio.

## Build Status
- npm install: pass
- npm test: pass (111)
- npm run build: pass
- Local preview: pass
- Production verification: pass

## Deployment
- Repo created: yes — https://github.com/ben-gy/au-mortality
- GitHub Pages enabled: yes (`build_type=workflow`)
- Cloudflare DNS CNAME created: yes
- TLS cert: issued after one CNAME cycle (`https_enforced=true`)
- Deploy workflow: success
- PR created: https://github.com/ben-gy/au-mortality/pull/1
- Live bundle vs local `dist`: identical (`index-myB5gKta.js`)

## Errors & Resolutions

**Pipeline**

1. *En dashes vanished* — the pooled period rendered as `20202024` and ICD ranges as
   `I20I25`. The MORT CSVs are CP1252; `buf.toString('latin1')` maps byte 0x96 to a C1
   control character that then disappears from the DOM. Node ships no `cp1252` decoder and
   `TextDecoder('windows-1252')` needs a full-ICU build, so the 32 C1 slots are mapped
   explicitly in `parse.mjs` (with the five unassigned ones → U+FFFD) and unit-tested.
2. *Boundary file 2.66 MB* — above the 1 MB target. Raised mapshaper simplification to 1.2%
   and dropped the 17 non-spatial SA3s ("Migratory - Offshore - Shipping", "No usual
   address") that have no mortality row and would have rendered as unexplained grey
   slivers. Result: 340 polygons, 0.86 MB.

**Frontend — four defects caught during verification, none shipped**

3. *Comparator line off-panel* — `barChart`'s reference marker took a percentage of the
   whole chart element instead of the bar track, so the "national" line drifted right and
   fell outside the panel entirely on the socioeconomic gradient. Now computed from
   `100% − label − value − gaps`.
4. *"Palmerston ranks 323rd of 322"* — `rankFlips` ranked each measure over everything
   published (different denominators, since a region can have a crude rate published and its
   standardised rate withheld) and the caller applied its population floor *afterwards*.
   Now the eligible set is settled first — caller filter, then both values present — and
   ranks are assigned only then. Guarded by three tests.
5. *Explorer sorted backwards* — `dir` was `+1` for `'asc'` and the comparator was
   `(bv - av) * dir`, so "descending" produced ascending order while the header caret said
   ▼. The default view opened on Australia's *healthiest* regions while claiming to rank by
   avoidable deaths. Extracted as a pure `sortRegions()` and tested.
6. *"0.42× difference end to end"* — remoteness runs best-to-worst but socioeconomic
   quintiles run worst-to-best, so a fixed `last/first` ratio reported one gradient upside
   down. Now always states the larger over the smaller with the matching names.

Also fixed: a stale tooltip could strand itself on screen (a removed element never fires
`mouseout`), so `hideTooltip()` now runs on view change, modal open, drawer open and scroll;
colliding slope-chart labels were de-collided with a tested `spreadLabels()` plus leader
lines; and accessible names were added to bar rows, histogram bins and clickable table rows,
which had been focusable with no announced label.

**Verification environment**

The Browser pane's screenshot capture intermittently returned blank or offset images while
the page was scrolled — a pane artifact, confirmed against the DOM each time rather than
assumed. Working at a 1280×2000 viewport put whole views inside one capture and made
screenshots reliable. Real-click verification used trusted pointer events throughout
(`computer` clicks by ref and by coordinate); `element.click()` was never used to validate
an interaction.

## Follow-ups for the user

- **`au-insolvency` still has no TLS certificate** — three days after its 2026-07-19 build.
  I checked rather than assumed: `https_enforced` is still `false` and
  `https://au-insolvency.benrichardson.dev/` fails with an SSL error (curl exit 60). I ran
  the documented CNAME-cycle remedy once more this run and it did **not** resolve within 45
  seconds, so its registry status stays `verify_production_pending`. This is now beyond
  normal propagation and likely needs a look: check that the Cloudflare CNAME for
  `au-insolvency` exists, is **grey-cloud / DNS-only** (a proxied orange-cloud record blocks
  Let's Encrypt's HTTP-01 challenge and is the usual cause), and that GitHub Pages shows no
  domain-verification error. Every other site built since has certified normally, including
  this one, which points at that specific DNS record rather than anything systemic.
