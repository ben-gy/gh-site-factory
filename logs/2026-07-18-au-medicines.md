# Build Log: Prescription Medicines
**Date:** 2026-07-18
**Status:** deployed

## Idea Source

Researched. `IDEAS.md` was empty (no lines starting with `-`), so I went looking for a domain the
catalogue doesn't cover. Existing AU sites already span income, solar, MP expenses, immunisation,
super, treaties, jobs, pokies, pay gap, contracts, government structure, flights, childcare,
charities, pollution, aged care, crime, planning, crashes, donations, hospitals and dams —
pharmaceuticals were a genuine gap, and a different domain and audience from `au-hospitals`
(hospital performance) and `au-agedcare`, so it is a new product rather than an EXPANSION_IDEAS entry.

Search path: `data.gov.au` CKAN returned nothing usable for PBS (the department publishes outside the
portal). A web search located the PBS "Date of Supply" statistics page, and probing it directly found
real machine-readable bulk CSVs — six financial-year files plus an item→drug map — with no auth and no
API key.

Deliberately rejected during selection: ACARA school data (licence-restricted, not openly
downloadable) and ATSB aviation occurrences (too close to the existing `au-flights`).

## Site Details
- **Name:** Prescription Medicines
- **Repo:** ben-gy/au-medicines
- **Category:** health
- **Audience:** health journalists on deadline, pharmacy/health-policy analysts, and general readers
  arriving from a story about a specific drug
- **Stack:** Vanilla TypeScript + Vite 6 + Vitest. No chart library; all SVG hand-rolled.
- **Data strategy:** pipeline. Cron is **monthly** (`23 4 9 * *`), matching the source: the department
  republishes the entire Date of Supply report each month, adding one month and restating earlier ones
  as late pharmacy claims arrive. Monthly is both the correct proportional cadence and the factory's
  fastest permitted one. Day 9 / 04:23 UTC to stagger against the other repos.

## Data Sources

- PBS and RPBS Section 85/100 **Date of Supply supplementary reports** — 6 financial-year CSVs,
  ~186MB total: https://www.pbs.gov.au/statistics/dos-and-dop/dos-and-dop
- **PBS item drug map** (item code → generic name, form/strength, ATC5), 12,119 items:
  https://www.pbs.gov.au/statistics/dos-and-dop/files/pbs-item-drug-map.csv
- **WHO ATC level 1/2 nomenclature** — embedded in `pipeline/atc-names.mjs`. The PBS data carries ATC5
  codes with no names attached, so the 14 anatomical groups and ~94 therapeutic subgroups are supplied
  from the standard (stable, tiny) terminology rather than scraped.

Scale: 2,047,223 source rows → 1,013 medicines × 69 months (Jul 2020 – Mar 2026). 1.88bn prescriptions,
$68.27bn Commonwealth subsidy, $18.28bn patient contribution.

## Architecture Decisions

**Vanilla TS over React.** Single page, eight tabbed views, no routing beyond a `#drug=` hash and no
deeply nested component state. 84KB of JS (27KB gzipped) for the whole application.

**No map, deliberately.** This dataset has no geography — not a postcode, LGA or state column anywhere.
Three of the last four builds were choropleths; adding a fourth here would have meant inventing a
geographic dimension that does not exist. Noted because "no map" is a conscious choice against the
house default, not an omission.

**No force-directed network, deliberately.** Medicines do not connect to each other. A relationship
graph would have been borrowed decoration.

**Sunburst instead of a treemap for the hierarchy.** The ATC classification is a true 3-level tree
(14 → ~94 → 1,013). A radial partition preserves the ring-per-level reading of "which body system, then
which drug class"; a treemap flattens that. Written as pure geometry in `src/utils/sunburst.ts` with
positional tests.

**Log-log scatter for cost vs volume.** Cost per prescription spans four orders of magnitude and volume
spans seven; no linear chart survives that. Equal-total-spend diagonals turn the scatter into a budget
chart.

**Money model — the trap in this dataset.** `GOVT_CONTRIB + PATIENT_CONTRIB == TOTAL_COST` holds only
for above-co-payment rows. For under-co-payment rows the department fills `PATIENT_CONTRIB` with the
*maximum schedule price*, not what anyone paid; summing it overstates patient spending by roughly
$300m/year ($1,912m notional vs $1,604m actual in FY2024-25 alone). The pipeline uses
`PATIENT_NET_CONTRIB` (actual, collected since Jan 2016) throughout, and the About modal explains the
choice. This was caught by profiling the fields before writing the aggregator rather than after.

**Discovery by content pattern, not a hard-coded file list.** `collect.mjs` parses the index page for
`dos-*-phrmcy-type.csv`, so the pipeline does not silently go stale when the financial year rolls over
each July. The department's WAF rejects non-browser user agents, so the collector sends a browser UA.

## Per-view UX critique

| View | Question it answers | Hover | Click | Next action |
|---|---|---|---|---|
| Rankings | What dominates, by money or volume? | Exact scripts, gov/patient split, cost per script, class | Row → drill-down | Switch metric; the top-10 changes completely |
| Explorer | What's the number for the drug I came for? | Per-cell values, YoY window comparison | Row → drill-down | Sort/filter/search |
| Hierarchy | Where does the money sit in the therapeutic tree? | Value + share of group | Ring → zoom; leaf → drill-down; side list → same | Zoom in, then out via hub or breadcrumb |
| Cost vs volume | Expensive because everywhere, or expensive per dose? | Full profile per dot | Dot → drill-down; legend → isolate group | Zoom/pan into the dense cluster |
| Movers | What actually changed? | Both window totals + multiple | Line → drill-down | Switch scripts/cost |
| Trends | What did the policy changes do? | Group, month, value, share | Segment or legend → isolate group | Switch metric; read the event markers |
| Who pays | Who carries which body system? | Cell value + row share | Self-funded row → drill-down | Switch metric |
| Insights | What should I have asked? | — | Card → medicine or view; histogram bar → list its medicines | Follow the card |

Every interactive-looking element is wired: no dead clicks. Legend swatches on the scatter and trends
are keyboard-focusable buttons, not decoration.

## Test Results
- Tests written: **109** across 5 files
- Tests passed: **109**
- Tests failed: 0 (after fixes)

Two failures on first run, both wrong expectations in my own tests rather than code defects: I asserted
9 sunburst descendants as 8, and `count(92_200_000)` as `92m` when the helper's threshold correctly
keeps one decimal below 100m. Corrected the tests.

Positional layout tests (per the hard rule, area-only assertions are insufficient):
- **Sunburst:** every ring tiles the full circle with flush seams (each arc starts exactly where the
  previous ended), children nest strictly inside their parent's span, equal radial thickness, in-bounds
  radii, no NaN, degenerate inputs return `[]` rather than broken geometry, full-circle arcs emit two
  sub-paths instead of one degenerate arc.
- **Label de-collision:** no two labels closer than the minimum gap, relative order preserved, all in
  bounds, tail pulled back when the forward sweep overruns, even distribution when they cannot all fit.
- **Zoom:** never zooms out past base, respects max scale, clamps panning inside bounds.
- **Pipeline:** the quoted-comma CSV trap (`"Capsule, 500 mg"`), doubled-quote unescaping, slug stability.

## Build Status
- npm install: pass
- npm test: pass (109/109)
- npm run build: pass
- Local preview: pass
- Data pipeline in CI: pass (ran on push, re-downloaded 186MB and committed refreshed data)

## Defects found during verification and fixed

1. **Colliding labels on the Movers slope chart.** Several medicine names and change figures printed on
   top of each other where a log axis stacked them within a few pixels. Fixed with a new
   `src/utils/declutter.ts` (marks stay at true positions, labels are pushed apart, leader lines
   connect them) plus 9 positional tests.
2. **Overlapping event annotations on Trends.** "Co-payment cut to $30" and "60-day dispensing begins"
   are eight months apart and overprinted. Fixed by alternating crowded labels onto a second row;
   verified programmatically that no two label boxes intersect.
3. **Ghost tooltip after a click.** Clicking a mark opened the drill-down over the cursor; with no
   pointer movement no `mouseout` fired, so the tooltip floated on top of the new panel. Fixed by
   hiding on click in the capture phase.
4. **`1880m prescriptions`** in the About modal — `count()` had no billions branch. Added one.
5. **Dead whitespace** in the Who pays left column. Filled with a second, genuinely informative split
   (subsidised vs under-co-payment by volume) that makes the volume-vs-cost point directly.
6. **Misleading insight copy.** "Fluorometholone prescriptions have fallen 100%" — a fall to exactly
   zero is a delisting from the Schedule, not collapsing demand. Reworded for the zero case, with a
   test, and the Movers legend now says so too.

## Verification performed

**Local (pre-deploy):** all 8 views rendered; no horizontal overflow at 375px asserted programmatically
in every view *and* with the drill-down open (the PBS items table scrolls locally inside its
`overflow-x: auto` container, so the page itself never scrolls sideways); zero console errors.

**Production (`https://au-medicines.benrichardson.dev`, mandatory):**
- Deployed bundle filename matches the local `dist/index.html`; `/data/drugs.json`, a per-drug chunk and
  `/og.png` all 200; HTTP→HTTPS 301; `https_enforced: true`.
- All 8 views checked for content, NaN/undefined text and emptiness — all clean, zero console errors.
- **Real clicks only** (Chrome MCP pointer events, never `element.click()` for interaction verification):
  rankings bar → drill-down; sunburst arc → zoom with breadcrumb; scatter dot → drill-down + hash;
  trends segment → group isolation; histogram bar → bin listing (40 medicines); glossary link → popover.
- **Drag-vs-click on the zoomable scatter:** at 1× a drag correctly does nothing (already clamped to
  base); zoomed in, a drag pans the viewBox and does **not** fire a click, and a plain click on a dot
  while zoomed still opens the drill-down. The deferred-pointer-capture pattern behaves correctly.
- **About modal opened from over the sunburst** (the z-index case that has shipped broken before) —
  screenshotted, sits fully above the visualization.

Note on screenshots: the browser pane returns a mis-composited frame when the page is scrolled (header
painted low, large blank region). Confirmed a capture artifact, not a defect — the DOM geometry was
correct throughout (`headerTop: 0`, consistent document/element heights) and the same content renders
perfectly at `scrollY: 0`. Worth remembering for future runs so it isn't chased as a layout bug.

## Errors & Resolutions

- **`gh api ... -f https_enforced=true` returned 422** ("not of type boolean"). `-f` sends strings; `-F`
  sends typed values. Re-ran with `-F` and the certificate flag flipped to true.
- **`git push` to main rejected** after the fix commit — the Data Pipeline workflow had already run on
  push and committed refreshed data. `git pull --rebase` then push.
- **`preview_start` failed** on a stale `.claude/launch.json` at the worktree root pointing at a
  non-existent `2026-07-13-au-visas`. Served the production build with `vite preview` and opened it by
  URL instead.
- **Browser served a cached JS bundle** when re-verifying the reworded insight, showing the old copy
  even though the deployed asset contained the fix (confirmed with `curl | grep`). Re-navigated with a
  cache-busting query to confirm on the live site.
