# Build Log: Bushfires
**Date:** 2026-08-03
**Status:** deployed

## Execution mode

Ran under **ultracode on Opus 5**. Multi-agent orchestration was used for the two phases
where breadth and independent judgement pay — candidate research (6 parallel agents, each
required to *fetch* its sources rather than recall them, plus a judge) and adversarial
verification of the shipped claims (6 skeptics + a judge). Every subagent was pinned to
`model: 'opus'`. All irreversible plumbing — `git init`, `gh repo create`, DNS, Pages,
the PR — was run serially in the main loop.

## Idea Source

`IDEAS.md` was empty, so a new idea was researched. Six candidates were investigated in
parallel, each agent bound to actually fetch and parse its sources:

| Candidate | Outcome |
|---|---|
| **au-bushfires** | **WON** — CC BY 4.0, real polygon geometry, a live client-side layer, and a falsifiable external anchor |
| au-agriculture | Runner-up. Excellent ABS SDMX data at SA2; loses on repeat-visit pull and view diversity. **Recommended for the next run.** |
| au-schools | **Disqualified on licence.** ACARA's School Profile/Location files are governed by the My School terms of use, clause 6.4 of which prohibits reproducing the content "on a publicly accessible website". Data quality was arguably the best of the six; it is simply not ours to republish. Do not build without written ACARA permission. |
| au-work-safety | Clean CC BY 4.0 but state-only geography (8 polygons), annual, no live component — would render as a good set of matrices and bars, the exact shape the feedback file warns against. |
| au-air-safety | ATSB states in writing that it does not endorse commercial use of its occurrence data and offers no bulk export; what remains needs ~7,400 fragile HTML fetches. |
| au-energy-plans | Technically excellent and CORS-open, but duplicates an official AER tool, has a genuinely grey licence, and carries zero history. Logged to `EXPANSION_IDEAS.md`. |

## Site Details
- **Name:** Bushfires
- **Repo:** ben-gy/au-bushfires
- **Category:** environment (index category `data-explorers`, country `AU`)
- **Audience:** People who live where it burns (phone, October, often anxious); journalists
  in fire season; researchers and planners
- **Stack:** Vite 6 + vanilla TypeScript + Leaflet 1.9 + Vitest
- **Data strategy:** **pipeline** — quarterly cron (`23 4 9 2,5,8,11 *`), proportional to
  the source's irregular ~annual republication, timed so a refresh always lands before the
  southern fire season. Plus **client-side realtime** for the one live view: the browser
  polls the public CORS-open feeds directly, and **no scheduled Action ever touches live
  data or archives it**.

## Data Sources
- Digital Atlas of Australia — Historical Bushfire Boundaries v2.0 — 311,984 polygons,
  1899-12-30 → 2023-10-15, seven states (CC BY 4.0, verified from the ArcGIS item's own
  `licenseInfo`, owner `aus_digitalatlas`)
- Digital Atlas — Historical Bushfire Extents 2020–25, layer 3 "Other States" (43,778) and
  layer 0 "Northern Territory" (2,489 — the only NT data that exists anywhere)
- Digital Atlas — Near Real-Time Bushfire Extents (511 features, ~3-hourly, client-side)
- NSW RFS `majorIncidents.json` (client-side)
- ABS ASGS 2021 LGA (566) and state boundaries, CC BY 4.0

## Architecture Decisions

**Vanilla TS, not React.** One page, nine views, no nested component tree.

**Server-side spatial intersection for the regional view.** "Has my area burnt" needs
administrative geography the fire data does not carry. Rather than approximate by centroid
— which would attribute a 200,000 ha fire wholly to whichever council contained its middle
— each of the 547 ABS council polygons is POSTed back to the fire service as a spatial
filter, so the authoritative service computes the intersection. 6 KB of geometry per
request, ~0.2 s each, ~10 minutes for the pass.

**Geometry budget.** Server-side `maxAllowableOffset` alone still left 1.9M vertices across
7,260 polygons (38 MB). mapshaper in the pipeline takes that to 3.5 MB raw / ~800 KB over
the wire with every fire and its recognisable shape kept, lazy-loaded only when the map
opens.

## Data correctness — the substance of this build

**The multi-part trap.** A fire made of several disjoint burnt patches is stored as one row
per patch, and **every row repeats the parent fire's full `area_ha` and `perim_km`**. This
was verified directly rather than assumed: WA's `BF 2019 ESP 013` returns 23 rows, all with
`area_ha = 162367` and `perim_km = 1290`, while their individual `Shape__Area` values differ
by up to 42× and **sum to ≈162,000 ha** — exactly the figure each row claims alone. That is
positive proof the rows are parts of one fire, so keeping one row per fire is precisely
correct.

Naive 2019-20 total: **31.8 Mha**. Corrected: **16.5 Mha** (1.93×).

**Dedupe key:** `(state, fire_id, ignition_date, area_ha, perim_km)`. `fire_id` alone is not
a key — there are nulls, literal `'0'`s, and placeholder values like `999` reused across
unrelated fires.

**THE BIG ONE — and it shipped before it was caught.** The first version *never* merged rows
with a null or `'0'` fire_id, reasoning that two anonymous rows cannot be *proven* to be the
same fire. That rule was validated against the 2019-20 season, where it cost **46 ha**, and
looked obviously safe. It was not. The 2019-20 season simply has almost no id-less
duplicates, so the test was **unrepresentative of the archive**.

Across the whole file it left standing six South Australian records (OBJECTID 310689–310694)
that are byte-identical on state, ignition date (31 Dec 1980), area (**5,763,897 ha**) and
perimeter (7,118 km), differing only in geometry. Keeping all six added **28.8 million
hectares of fire that never happened** — 93.6% of a season — and put 1980-81 at the top of
the chart as *"the largest fire season on record, 36.9 Mha"*. That claim was published.
Corrected, 1980-81 burnt **8.12 Mha** and ranks fifteenth; the largest mapped season is
**2012-13 at 24.3 Mha**, and Black Summer ranks **4th**, not 5th.

Found by the adversarial verification pass, which was told to *refute* the site's headline
rather than confirm it. No test I had written could have caught it: every gate I built
measured the season totals the bug produced.

The rule now: id-less rows are still de-duplicated on the remaining four fields, because two
distinct fires cannot share an area to the hectare **and** a perimeter **and** a date. Only
zero-area rows stay unmerged — they cannot double-count area, and collapsing them would
destroy fire counts for no benefit.

**The layer-overlap trap.** The historic layer runs to 2023-10-15 and the 2020–25 layer
starts 2020-07-01 — they overlap for three seasons and hold the *same* fires (VIC 2021-22:
2,046 rows in one, 2,066 in the other; SA 176 vs 176). Union them and you double three
seasons. Cutover is 1 July 2020, asserted in the pipeline.

**The silent-zero projection trap.** The ABS boundary service returns **Web Mercator
(3857)**, not the 4283 its sibling services use. Labelling that geometry `inSR=4283` makes
ArcGIS read metres as degrees and match nothing — returning `count: 0` for every council,
with no error. Found only because Blue Mountains obviously should not have zero fires.

**The pagination trap.** The historic layer honours `resultRecordCount=2000`; the 2020–25
layers cap at 1000. Breaking on `feats.length < PAGE` therefore stopped after one page and
silently lost 45,000 of the newest records. Now advances by the count actually returned and
trusts `exceededTransferLimit`. **Caught by the anchor gate, not by inspection.**

### Anchor gates (external, falsifiable, enforced twice)
1. Deduped NSW 2019-20 ∈ [5.30, 5.65] Mha vs the NSW Bushfire Inquiry's 5,520,000 ha →
   **5,492,279 ha** ✓
2. Largest ACT fire 2019-20 ∈ [84k, 92k] ha vs Orroral Valley's published ~87,000 →
   **Orroral Rocks Wildfire, 87,926 ha** ✓
3. Naive/corrected ratio ≥ 1.9 → **1.93×** ✓
4. No historic-layer record at or after the 2020-07-01 cutover ✓
5. **No duplicate (state, date, area, perimeter) group above 1,000 ha may survive the merge**
   implying more than 0.1% phantom area — added after the failure above ✓ (1 group tolerated,
   3,766 ha against a 341,879 ha cap; the tolerance is logged, never silent)
6. **No single fire may exceed 85% of its own season** — the SA record was 97.4%, which no
   chart would ever have made obvious ✓

Gates 1 and 2 check **accuracy**; only 3, 5 and 6 check the **correction**. That distinction
is not pedantry: de-duplicating NSW moves it by 337 ha (0.006%), so the NSW anchor would have
passed happily while the archive carried 28.8 Mha of phantom fire. The site now says so
explicitly rather than implying the anchors prove more than they do.

Enforced in `pipeline/collect.mjs` at harvest time **and** re-asserted against the committed
JSON in `tests/data.test.ts`, so a bad file cannot ship even if committed by hand.

## Per-view UX critique

| View | Question answered | Hover | Click | Next action |
|---|---|---|---|---|
| Overview | What am I looking at, what's surprising? | stat tooltips | finding cards → their view | jump into any view |
| Your area | *Has my council burnt, how often, when?* | per-cell caveats | row → drawer with 94-bar season sparkline | compare councils, change ranking metric |
| Map | Where are the scars? | every polygon via `bindTooltip` | — (pan/zoom) | filter by type/season |
| Seasons | Which years burnt most? | full per-state breakdown per column | column → season drawer | see the top-ten table |
| Fire size | How big is a typical fire? | count + area + share per bucket | — | cross-read against the state matrix |
| Causes | What starts fires, and what do we know? | per-cell values | matrix cell → cause × state drawer | switch fires/area measure |
| Coverage | Is WA fire, or WA mapping? | per-bar detail | bar → state profile drawer | compare raw vs per-10,000 km² |
| The correction | Can I trust the number? | per-duplicate detail | — | read the rebuilt Black Summer table |
| Right now | What's burning? | per-incident tooltip | — | read the typed breakdown |

No dead clicks: every `cursor:pointer` affordance is wired, and non-interactive marks carry
no pointer affordance.

## Test Results
- Tests written: **71** (parse 29, shipped-data 25, layout 17)
- Tests passed: **71** / failed: 0
- `tests/layout.test.ts` asserts **positions** on the real chart layouts — in-bounds, no
  pairwise overlap, flush stacking, monotonic log heights, no NaN, degenerate inputs. The
  copied treemap template and its `squarify.ts` were **deleted** rather than kept as dead
  code, and replaced with tests for the layouts this site actually renders.

## Build Status
- npm install: pass
- npm test: pass (71)
- npm run build: pass
- Local preview: pass
- Production verification: pass

## Deployment
- Repo created: yes
- GitHub Pages enabled: yes (`build_type=workflow`)
- Cloudflare DNS: created (`au-bushfires` CNAME → `ben-gy.github.io`, DNS-only)
- TLS certificate: **approved** on the 2nd poll (polled `.https_certificate.state`)
- PR created: https://github.com/ben-gy/au-bushfires/pull/1
- Workflow triggered: yes — Deploy succeeded; the Data Pipeline also ran on push, which
  exercises the clean-runner path (`cd pipeline && npm install` for mapshaper, never
  `npm ci`)

## Errors & Resolutions

1. **Pagination truncation** — 2020–25 layers cap at 1,000 records, not 2,000; the first
   harvest silently lost 45k rows. Caught by the completeness gate. Fixed to page on the
   returned count + `exceededTransferLimit`.
2. **Bad completeness threshold** — the gate compared *deduped fires* against a *raw-row*
   expectation and failed a perfectly good run. Split into two separate checks.
3. **Synthesised data in the causes matrix** — area-by-state was initially estimated as
   state count × national mean size. That is inventing data; the pipeline now computes real
   `areaByState` and the estimate was removed.
4. **Whitespace-only fire names** — the source uses `" "` as well as `null` for unnamed
   fires. `" "` is truthy, so it slipped past every `name ? name : '—'` fallback and
   rendered as an empty table cell that read as a rendering bug. Fixed at source with
   `cleanName()` plus a client-side guard.
5. **Live map projected against a stale container size** — `invalidateSize()` on a 60 ms
   timer after layers were added left the shared canvas renderer holding paths projected
   against the old size, drawing the state outlines hundreds of kilometres from the markers.
   Fixed by sizing before projecting and using `animate: false` on `fitBounds`. The same
   latent race was fixed pre-emptively in the main map view.
6. **Live extents drawn as centroid dots** — a 40,000 ha burn scar as a 5 px dot is both
   the wrong encoding and invisible at continental zoom, so the map read as empty while the
   counters said 467. Now draws the real polygons and frames the view on actual activity.
7. **Close controls 13×22 px** — every overlay's ✕ was well under the 44×44 minimum. Fixed
   for drawer, modal and glossary popover.
8. **Stale bundle during verification** — several "still broken" readings were taken against
   a cached bundle three fixes old. Caught by comparing the loaded `<script src>` against the
   built hash; all conclusions were re-derived from the current bundle.
9. **Research claim disproved** — the research brief reported the VIC Emergency feed as
   `CORS *`. A header check with an explicit `Origin` showed it sends **no**
   `access-control-allow-origin` at all and cannot be used from a browser. It was dropped
   and its absence is explained in the About panel. Conversely the NSW RFS feed emits
   `ACAO: *` **only** when `Origin` is present — a bare curl makes it look unusable.

## Adversarial verification (the phase that earned its keep)

Six skeptics, each pinned to Opus 5 and each told to **refute** one of the site's central
claims against primary sources, plus a judge. Results:

| Claim | Verdict |
|---|---|
| NSW Black Summer = 5.49 Mha vs published ~5.5 Mha | **survived** — reproduced independently from the service; the published figure confirmed as NSW-specific from the NSW Bushfire Inquiry Final Report, Table 2-2 (5,520,000 ha) |
| The 1.93× overcount and the 23-copy WA fire | **survived** — every figure reproduced from the primary service |
| The NT is absent from the historical layer | **survived** — attacked on every axis; seven state values summing to the full row count, `LIKE '%NT%'` returns 0 |
| CC BY 4.0 licensing and attribution | **survived**, but surfaced four real compliance gaps |
| 10.9% cause coverage; lightning bigger than deliberate | **survived**, but surfaced a **factually false caption** |
| **Black Summer ranks 5th; 1980-81 largest at 36.9 Mha** | **REFUTED** — see above. The single most valuable finding of the run. |

Two of these would have shipped as published falsehoods without this phase:

1. **The 1980-81 record season was an artefact of my own under-merging.** Detailed above.
2. **A false caption on the Causes view**: *"prescribed burns should almost all be recorded
   as prescribed burning, and they are."* Zero of 83,621 prescribed-type fires carry that
   cause; 99% carry none at all. The caption stated the exact opposite of the chart beneath
   it. It now states the real — and more interesting — finding: the cause field records what
   agencies **investigate**, not what happens.

Three further corrections came out of it:

3. **An overstated methodological claim.** The site said the NSW and Orroral anchors meant
   "if a change breaks the correction, the build fails". False: de-duplicating NSW moves it
   by 337 ha, so a broken dedupe sails straight past both. The copy now separates accuracy
   checks from correction checks and names the gate that actually guards the dedupe.
4. **"Largest season on record" was never defensible**, even after the fix — the NT is absent
   before 2020 and 1974-75 (~117 Mha nationally, ~15% of the continent) appears as 2.3 Mha.
   Every ranking now says "in this data", and the Seasons view carries a standing warning
   against reading the pre-satellite era as history.
5. **CC BY 4.0 compliance**: the licence requires a statement of changes, which was missing.
   Added, along with the ABS and NSW RFS copyright notices, the RFS's requested pointer to
   rfs.nsw.gov.au for current information, and a no-endorsement line.

The mean-vs-median problem was also caught: 31% of "deliberate" records carry zero area and
one 1.7 Mha fire dominates the average, so the site now quotes the **median** (13×) instead
of the mean (4×).

## Things worth remembering
- The `[hidden] { display: none !important; }` guard was **missing** from the au-ip
  stylesheet this site was seeded from (au-ip is not broken by it because its overlays
  attach/detach rather than toggling the attribute). Added here verbatim.
- The Digital Atlas 2020–25 service path contains a literal U+2013 EN DASH that must be
  percent-encoded as `%E2%80%93`, or it resolves locally and 404s on CI.

## Tone note
Bushfires killed 33 people directly in 2019-20 and destroyed over 3,000 homes. There is no
gamified microcopy, no "leaderboard of destruction", and no playful loading text anywhere on
the site. The 2012-beat-Black-Summer finding is framed as *area burnt and public memory are
decoupled because remote fires destroy no homes* — never as "Black Summer wasn't that big".
The live view carries an explicit instruction to use state emergency services for warnings.
