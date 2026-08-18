# Build Log: Native Title
**Date:** 2026-08-19
**Status:** deployed

## Idea Source

Researched. `IDEAS.md` was empty, so six candidate domains were investigated in parallel — AFCA
financial complaints, ATSB aviation safety, native title, BOM tropical cyclones, ABS criminal courts,
and GrantConnect federal grants — each by an agent required to verify every source with a real fetch
and to report honestly what it could not verify. A separate judge scored them.

**Winner: native title (87).** Runner-up GrantConnect (84), logged below as the next build.

Three candidates were rejected on grounds that would not have shown up without doing the fetching:

- **AFCA** — its terms prohibit redistribution in writing, there is nothing on data.gov.au, and the
  only reachable endpoint is a typeahead capped at 10 results. Licence-blocked outright.
- **ATSB** — the premise was false. There is no bulk file; the publisher states in writing that the
  database is not downloadable, and the data portal is NXDOMAIN.
- **BOM tropical cyclones** — I had independently confirmed the file was live and superb (31,706
  track fixes, 1,133 systems, 1907→2026, 7.9 MB, refreshed the day of the run). It was rejected
  anyway because the dataset page states **no licence**, and bom.gov.au's copyright page grants
  CC BY 4.0 only where explicitly stated. Republication rights could not be established. This is the
  clearest argument for having run the study rather than building the first thing that fetched.

## Site Details
- **Name:** Native Title
- **Repo:** ben-gy/au-native-title
- **Category:** government-transparency
- **Audience:** native title practitioners; land councils and body-corporate administrators; tenement,
  heritage and infrastructure consultants doing due diligence; journalists and researchers; and
  members of the public looking up a place they know
- **Stack:** Vanilla TypeScript + Vite 6, Leaflet 1.9 (only runtime dependency), hand-rolled SVG
- **Data strategy:** pipeline, **monthly cron** (day 6, 16:23 UTC). The source refreshes daily, but
  determinations, agreements and applications land a few dozen a year — monthly is both the factory's
  fastest permitted cadence and genuinely proportional to the data.

## Data Sources

All seven registers are published by the National Native Title Tribunal via data.gov.au, each with an
explicit `license_id: cc-by-4.0` on its CKAN record, all extracted 2026-08-17/18:

| Register | Rows |
|---|---|
| National Native Title Register (determinations) | 683 |
| Native Title Determination Outcomes | 1,162 |
| Register of Native Title Claims | 96 |
| Schedule of Native Title Determination Applications | 189 |
| Indigenous Land Use Agreements | 1,539 |
| Registered Native Title Body Corporate areas | 296 |
| RATSIB boundaries | 18 |

Plus Geoscience Australia's published land areas (the denominators) and the fleet's ABS state
boundaries (used only to validate the projection).

**Deliberately excluded:** the Tribunal's four future-act layers (5,598 determination applications,
5,138 objections, 10,605 current notices, 384 s31 agreements). They exist only on the ArcGIS service;
their only data.gov.au entries are third-party mirrors marked `notspecified`. Their licence could not
be established to the same standard as the seven, so they are not republished — even though they
carry the most striking finding in the collection (of 5,598 future-act applications since 1994, none
resulted in a determination that the act must not be done). Logged to EXPANSION_IDEAS.md instead.

## Architecture Decisions

**Why the data.gov.au WFS and not the ArcGIS service.** The Tribunal publishes the same registers
twice. The WFS copy wins on both counts that matter: it has an explicit CC BY 4.0 CKAN record, and it
carries `JUDGE`, `NTHOLD`, `SEADET`, `DETBODY`, `APPEALDESC` and the whole ILUA statutory flag set,
none of which the ArcGIS copy exposes. Half this site would not exist on the other source. The cost
is that GeoServer answers 403 without browser headers.

**Why every area is dissolved.** The registers' polygons overlap, and the size of that error is not
uniform — which is the part that catches people out. On the outcome register the polygons were built
to tile and on land are nearly disjoint (0.02% to 3.4%). On the agreements register it is
catastrophic: the area column sums to 5,163,512 km² for ground that actually covers 2,434,646 km².
You cannot tell which case you are in by looking, so everything is a real union (`mapshaper
-dissolve2`) measured in Australian Albers.

**Why the map disables zoom animation.** See below — it shipped a bug that took an hour to find.

## Reconciliation Gates (11, all fatal)

| Gate | Result |
|---|---|
| Controlled vocabularies unchanged (9 fields) | pass |
| Registers link cleanly | 484 corporation→determination links |
| Projection reproduces Geoscience Australia's state areas | **0.09% drift** |
| The dissolve demonstrably ran (topology, not area) | 1,054 polygons → 4 unions |
| No union exceeds the sum of its parts | pass (overlap: exclusive 0.0%, non-exclusive −0.0%, does not exist 3.4%, extinguished 0.0%) |
| Agreements overlap is real and large | 5,163,512 km² summed vs 2,434,646 km² covered — 53% double-counted |
| Recognised decomposes into its two subsets | 0.004% |
| No jurisdiction has more native title than land | pass |
| Outcome register partitions the determination register | pass, 2 compensation matters explained |
| Findings do not contradict tenure polygons | pass |
| 35 cumulative dissolves land on the unfiltered total | 0.000% |

## Two false headlines the gates caught

Both were the same mistake — **comparing two sides of a ratio computed over different rows** — and
both came in from the research, not from the data:

1. **"46% of the outcome register is overlap."** The 756,812 km² in that claim was an all-rows sum
   including 332,894 km² of sea determinations, compared against a land-only union of 409,656 km².
   Like for like, the real figure is 3.4%. The site would have taught a fabricated number as its
   worked example.
2. **"Agreements sum to 74% more than the continent."** The 5,720,415 km² was again all-rows;
   land-only it is 5,163,512 km², comfortably inside Australia. The gate written to assert the
   rhetorically satisfying version failed immediately and was rewritten to assert the relationship
   that is actually true and actually load-bearing — that the union is far smaller than the sum.

`gateUnionBelowSum` now refuses to run at all unless a comparable sum is supplied for every class, so
the two sides cannot be sourced from different places by accident again.

## The map bug

The map shipped from the view agent rendering **exactly one zoom level too far out** — 219° of
longitude across the viewport instead of 110°, Africa on the left, Australia a small shape near the
bottom edge. Nothing errored. `getZoom()` was truthful, zoom-4 tiles were requested, and every one of
them was drawn at 128px instead of 256px.

The cause: Leaflet animates a zoom by putting a `scale()` on `.leaflet-tile-container` and clearing
it on `zoomend`. The deferred layout pass fires `invalidateSize()` and then `fitBounds()` on a
container that has only just been given its size; that lands mid-animation, the animation never ends,
and the scale is never cleared. Only the tile container's transform gives it away. Fixed with
`zoomAnimation: false`, which makes the failure structurally impossible rather than merely unlikely.

A second, self-inflicted bug followed: the fit-verification I added to catch the first one demanded
the viewport span be under twice Australia's width, which a **correct** fit can never satisfy in a
2:1 container (Leaflet's `zoomSnap` rounds down to zoom 4, legitimately leaving ocean either side).
It retried forever. Loosened to "contains Australia and is under four times its width", which accepts
every correct fit and still rejects the 219° failure.

## Test Results
- Tests written: **126** across 4 files
- Tests passed: 126
- Tests failed: 0

`tests/gates.test.ts` (35) feeds each guard the exact failure it exists to catch — a no-op dissolve,
a wrong projection, a contradiction between the two outcome vocabularies, a shrinking cumulative
series, prose in an identifier column — and asserts the build stops. `tests/hygiene.test.ts` (28)
enforces the language ban, the verbatim rule, the no-ranking rule and the "never sum an area column
in the browser" rule as failing tests. Two hygiene tests failed on first run; both were test bugs
(the scanner was reading its own explanatory comments) and were fixed in the scanner, not by deleting
the explanations.

## Build Status
- npm install: pass
- npm test: pass (126/126)
- tsc --noEmit: pass (0 errors across ~8,600 lines)
- npm run build: pass — 427 KB JS / 126 KB gzip, 55 KB CSS / 14 KB gzip
- Local preview: pass

## Verification performed

- All 8 views render with no error state, no stuck loading, no `NaN`/`undefined`/`Infinity`, and
  thousands of `[data-tip]` marks.
- **Zero horizontal overflow at 375px on all 8 views** (`scrollWidth === clientWidth === 375`).
- **The modal dismissal contract, at 375px, with the About modal opened over the Leaflet map**:
  baseline clean (no leaking `[hidden]`, viewport centre is page content); ✕ with a real 44×44 hit
  area; mouse `pointerdown` on the scrim; Escape from the document; and a real touch
  `pointerdown`+`pointerup` sequence outside the panel with no double-fire reopen. All four asserted
  independently by DOM presence and computed style, re-opening between each. Body scroll lock
  released afterwards.
- **Overlay stacking over Leaflet**: three probe points over the map all return elements inside
  `.overlay-host` while the modal is open. Map frame `isolation: isolate; z-index: 0` contains
  Leaflet's highest internal z-index (800) below the overlay layer (2000/2100).
- **Real trusted clicks** (not `element.click()`), after calibrating the screenshot→CSS ratio at 1.6
  with a `pointerdown` probe: a click on the Leaflet **canvas** over the Pilbara opened the dossier
  for *Palyku and Palyku #2* (WCD2021/003) with verbatim holders, the Federal Court judgment link,
  the tenure mosaic and the legal note; a click on an **SVG rect** in the timeline opened
  *Adnyamathanha People Native Title Claim No. 3* (SCD2015/002). Both set the `?id=` deep link.

## Deployment
- Repo created: yes — ben-gy/au-native-title
- GitHub Pages enabled: yes (build_type workflow)
- Cloudflare DNS CNAME created: yes — au-native-title → ben-gy.github.io
- Directory entry live on main: yes
- Workflow triggered: yes

## Errors & Resolutions

| Problem | Resolution |
|---|---|
| 209 MB outcome geometry arrived truncated on 2 of 3 attempts, surfacing as "Unterminated fractional number in JSON at position 204160968" | Byte count checked against `Content-Length` before parsing, 6 retries; truncation now reports as truncation |
| mapshaper `-clean` silently dropped 5 of 96 claims and 24 of 189 applications (small polygons become zero-area slivers after simplification) | `-clean` removed from every shipped layer; feature counts asserted before and after |
| Two false overlap headlines inherited from the research | Caught by gates; both rewritten against like-for-like figures |
| Leaflet stuck zoom-animation scale rendering the map one level out | `zoomAnimation: false` |
| Over-strict fit verification retrying forever | Loosened to a threshold a correct fit can satisfy |
| `import.meta.env` failed `tsc` | Added `src/vite-env.d.ts` and `vite/client` types |
| `register.ts` fetched `outcomes.json` outside the shared loader, duplicating a 128 KB download | Added `loadOutcomes` to `data.ts` |

## Notes for next time

- The runner-up, **GrantConnect** (359,318 federal grants, $234bn, 9 years, CC BY 3.0 AU), is ready to
  build and comes with a fully documented trap list — including a **silent 50,000-row truncation that
  returns HTTP 200 with no warning**, and the fact that measuring publication lateness against
  Approval Date rather than Start Date turns a 6.4% breach rate into a 75.3% one.
- The screenshot channel in the browser pane was unreliable throughout this run (compositing with a
  large blank band, intermittent 30s timeouts). Programmatic DOM assertions were used as the primary
  evidence and are what the verification above rests on.
