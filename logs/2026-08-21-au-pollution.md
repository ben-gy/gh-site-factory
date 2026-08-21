# Build Log: Industrial Pollution (au-pollution)
**Date:** 2026-08-21
**Status:** not_deployed_duplicate — a healthy live site already occupies this repo and dataset

## Outcome in one paragraph

The run researched, designed, built and locally verified a complete NPI explorer, then discovered
at the deployment step that **`ben-gy/au-pollution` already exists and is live** ("Pollution Watch
(AU)", built 2026-06-23, HTTP 200, cert approved, deploys succeeding). Nothing was pushed to it.
The build sits unshipped at `sites/2026-08-21-au-pollution/`. What the run did ship: the live site
was **missing from `index/sites.json`** — live at its own URL but invisible in the public
directory — and has been added; and four verified correctness defects in the live version have
been written up in `EXPANSION_IDEAS.md` with reproducible figures.

## Idea Source

Researched. `IDEAS.md` was empty. A six-candidate research workflow (7 Opus 5 agents, ~756k tokens)
scouted NPI, ACARA school profiles, ACIC wastewater drug monitoring, AFCA complaints, broadband
performance and GrantConnect. Two were disqualified on licence/format (ACARA: My School terms of
use clause 6.4 forbids republication on a public website; ACIC: PDF-only). The judge picked the
National Pollutant Inventory 10/8/9/9 (data availability / audience / visual richness / novelty).

**The novelty score was wrong, and so was mine.** Both the judge and I checked
`gh repo list ben-gy --limit 200` and `index/sites.txt`; `au-pollution` appears in neither.

## The duplication check failed — root cause

- The org has **353 repos**. The routine's prescribed check is `gh repo list ben-gy --limit 200`,
  which silently truncates. `au-pollution` is present at `--limit 400` and absent at `--limit 200`.
- `index/sites.txt` and `index/sites.json` did not contain `au-pollution` either, so the second
  check missed it too.
- `registry.json` **did** have it, marked `deployed`, but the file is 284 KB and was never read in
  full — only searched for the new idea's keywords after the fact.

**Fix for future runs:** raise the limit to `--limit 500`, and grep `registry.json` by topic
keyword *before* the design step, not just the public index.

## Site Details (of the unshipped build)

- **Name:** Industrial Pollution
- **Repo (intended):** ben-gy/au-pollution — **taken**
- **Category:** environment
- **Audience:** people living near heavy industry asking "what is released near me"; plus council
  environmental health officers, community groups, local journalists and researchers
- **Stack:** Vanilla TypeScript + Vite 6, Leaflet 1.9 from npm; treemap, matrix, Sankey, trend and
  concentration charts all hand-rolled SVG
- **Data strategy:** pipeline, **annual** cron (`23 6 9 4 *`, 9 April) — the NPI publishes on or
  before 31 March each year, so annual is the proportional cadence

## Data Sources

| Source | URL | Licence |
|---|---|---|
| National Pollutant Inventory (7 CSVs) | https://data.gov.au/data/dataset/npi | CC BY 4.0 |
| ABS ASGS state boundaries | `patterns/geo/au-states.geojson` | CC BY 4.0 |
| Australian postcodes | matthewproctor/australianpostcodes | CC BY 4.0 |

1,008,823 emission records, 128,178 transfer records, 8,140 facilities (100% geocoded), 93
substances, 27 reporting years (1998-99 → 2024-25). Nothing capped or sampled.

## Verified findings (each reproducible from the source)

1. **The EET mapping is counter-intuitive and easy to invert.** `data-dictionary.csv`: 1=Mass
   balance, 2=Engineering calculations, **3=Direct measurement**, 4=Emission factors, 5=Approved
   alternative. A scout agent initially assumed slot 1 meant "measured" and produced a spectacular,
   false headline ("only 1.2% of Australia's air emissions are measured"); the true figure is
   **56.7%**, a 47× error. My own first draft had the same inversion and was corrected against the
   dictionary before any number was published.
2. **PM10 is 97.9% fugitive and only 1.1% directly measured** (99.3% from generic emission factors),
   against **carbon monoxide at 88.3% directly measured**. The pollutant that does most damage to
   lungs is the one Australia measures least.
3. **Railton Works (cement, TAS)** reported 2,330,015 t of CO to air in 2024-25 against a 25-year
   median of ~1,939 t — a factor of 1,202, almost exactly 1000×, i.e. a unit slip. It alone is 79%
   of the published national CO total; excluding it moves the national figure from 2,964,850 t to
   634,839 t, a **367%** difference.
4. **Cross-substance sums double-count by 3.3%.** "Total Volatile Organic Compounds" already
   contains the named VOCs — **1,760 facilities reported both in 2024-25** — and PM2.5 sits inside
   PM10. The live site's `air_total_kg: 99,701,962,416` was reproduced to the kilogram and contains
   3,147,543,987 kg of double-counted mass.
5. **Air must come from `air_total_emission_kg`.** In 1998-99, 35.5% of air rows carry a total with
   both components blank; reconstructing from point+fugitive understates 1999-2000 by **27.2%**.

## Architecture Decisions

- **No cross-substance mass total anywhere**, enforced by a build-failing gate. A kilogram of
  mercury is not a kilogram of carbon monoxide.
- **Nine reconciliation gates**, published verbatim on the site's Method page. The air-total gate is
  deliberately *inverted*: it fails if point+fugitive ever reconciles exactly, because that would
  mean the guard had stopped testing anything.
- **PII by allowlist, not denylist.** `street_address` is deliberately dropped — the smallest NPI
  facilities are sole traders and family farms whose address is a residence. The guard is tested by
  injecting a contact field that does not exist today.
- **192 lazily-loaded history chunks** keyed by `facilityId % 192`, so a dossier costs ~140 KB
  instead of the full 26 MB.
- `pipeline/parse.mjs` is dependency-free plain JavaScript so CI exercises the traps directly.

## Test Results

- Tests written: **186** across 5 files (parse, format, layout, overlay, hygiene)
- Tests passed: **186**; failed: 0
- Three failures during development were my own wrong expectations, not code defects; one of them
  (`2.33 kt` vs `2,330 t`) exposed a genuine readability problem and the unit threshold was changed.

## Build Status

- npm install: pass
- npm test: pass (186/186)
- npm run build: pass (tsc + vite, 241 KB JS / 40 KB CSS)
- Data pipeline: pass — all 9 gates green
- Local preview: pass

## Verification performed (local production build)

Every view walked with **real clicks** through the browser, never `element.click()`:

- All 9 views render with real data; zero console errors from the site's own code.
- **Modal dismissal contract**: all four exits asserted by *state* (panel detached from the DOM,
  not hidden) — ✕ (44×44 hit area), scrim pointerdown, Escape, and a real `pointerType:'touch'`
  sequence that must not reopen. Baseline `[hidden]` leak check: 0.
- **About modal opened from the map view** and screenshotted: paints fully above Leaflet
  (scrim z-2000, panel z-2100, map container `isolation: isolate`).
- **Mobile at 375px**: the page cannot be scrolled sideways on any of the 9 views
  (`window.scrollX` stays 0 after `scrollTo(400,0)`). Note that
  `documentElement.scrollWidth` is **not** a reliable proxy here — it counts content inside an
  internal `overflow-x:auto` scroller and reported a 740px "overflow" on the matrix view that does
  not exist. The matrix was still tightened for mobile (758px → 453px).
- Facility drawer opens and **detaches** on close (0 panels, 0 scrims), avoiding the iOS
  off-canvas trap.

## Defects found and fixed during verification

1. `2024–20` in the empty state — hand-rolled year slicing instead of the `financialYear` helper.
2. Searching "Port Pirie" returned **"BUNGAMA"** — the first locality in the postcode rather than
   the one that matched. The matched name now travels with the result.
3. `text-transform: uppercase` on the table head rendered **"≤10.0 µm" as "≤10.0 MM"** — micrometres
   silently became millimetres, a units error introduced purely by CSS.
4. `.toLowerCase()` on substance names produced "pm10" and "teq" mid-sentence; replaced with a
   prose helper that preserves acronyms, mirrored in the pipeline and tested for agreement.
5. Sankey destination **labels collided** where nodes were slivers (six labels in four pixels).
   Added a label de-collision pass with leader lines; rectangles stay exactly proportional. 14 new
   positional tests assert no two labels are closer than the gap.
6. Sub-pixel bars in the dossier read as "no data"; non-zero values now get a visible minimum.
7. An apparent dead click on the treemap was **my own coordinate error** — the click tool takes
   screenshot pixels (1.75× here), not CSS pixels. Measured the ratio with a pointerdown logger
   before concluding anything, per the standing note about this trap.

## Deployment

- Repo created: **no** — `ben-gy/au-pollution` already exists and is live
- Pushed: **no** — deliberately not force-pushed over a working deploy
- GitHub Pages: n/a
- **Directory entry live on main: yes** — the *existing* `au-pollution` was missing from
  `index/sites.json` and `index/sites.txt` and has been added
- Cloudflare DNS: n/a (already provisioned for the live site)

## Errors & Resolutions

| Issue | Resolution |
|---|---|
| Duplicate site discovered at deploy time | Stood down; nothing pushed. Root cause documented above. |
| Preview port 5199 and 5241 already serving other projects | Moved to 5731 after checking it was free |
| Browser pane scroll/screenshot intermittently stuck | Worked around with a tall viewport and region screenshots; all views verified |
| Stale bundle served after a rebuild | Caught by comparing the loaded bundle hash against the built one; hard-reloaded before re-verifying |

## Recommendation

Fold the four correctness fixes into the live `au-pollution` (they are cheap and two of them change
published numbers), or promote this build wholesale — it is complete, tested and verified. That is
the user's decision, not the routine's.
