# Build Log: Cyclone Tracks
**Date:** 2026-08-24
**Status:** deployed

## Idea Source
Researched. `IDEAS.md` was empty, so a 7-candidate research workflow ran on Opus 5 (one agent
per candidate, each required to actually download and parse the files rather than read a
landing page), followed by a judge.

- **BUILDABLE:** cyclones, grants (GrantConnect), airports (BITRE), atsb, homelessness (AIHW)
- **REJECT:** schools (ACARA — licence-blocked, independently reproducing the finding already
  recorded in `EXPANSION_IDEAS.md`), wastewater (ACIC — PDF-only across all 25 report cycles)

Winner: **cyclones**. Distinctness was the cleanest in the set — the fleet has no cyclone data
and no site whose entity is a moving trajectory, so this brings a visual language (scrubbable
track polylines) the catalogue has never shipped. 592 named storms are 592 long-tail search
terms, and the audience returns every November–April.

Runner-up **GrantConnect** (359,670 grant notices, ~$235bn, CC BY 3.0 AU) is genuinely strong
and should be built, but with a tighter brief — see the note appended to `EXPANSION_IDEAS.md`.

## Site Details
- **Name:** Cyclone Tracks
- **Repo:** ben-gy/au-cyclones
- **Category:** environment
- **Audience:** residents of the cyclone-exposed Australian coast; journalists; students
- **Stack:** Vanilla TypeScript + Vite 6, Leaflet 1.9 (npm), hand-rolled SVG charts
- **Data strategy:** pipeline, **monthly** cron (`23 4 9 * *`). IBTrACS is republished by NCEI
  periodically as seasons are reanalysed, not continuously — monthly is already faster than
  the source moves and is the fastest cadence the factory permits.

## Data Sources
- IBTrACS v04r01, Southern Indian basin —
  `https://www.ncei.noaa.gov/data/international-best-track-archive-for-climate-stewardship-ibtracs/v04r01/access/csv/ibtracs.SI.list.v04r01.csv`
- IBTrACS v04r01, South Pacific basin — same path, `ibtracs.SP.list.v04r01.csv`
- ABS ASGS state boundaries — `patterns/geo/au-states.geojson`, committed into `public/data/`

## The licence decision (the most consequential call of the run)

The obvious source is BOM's own `IDCKMSTM0S.csv`: same best tracks, friendlier shape, and it
carries observed storm-surge, wave and reported-wind fields IBTrACS does not. **It was
downloaded, parsed, verified — and then deleted and banned.**

A three-lens adversarial workflow (each lens instructed to *prove* republication was not
permitted) returned unanimously: **BOM FORBIDDEN, IBTrACS PERMITTED_WITH_CONDITIONS.**

BOM's copyright page, for content whose page states no terms of use:
> "Unless we state otherwise, you can download, copy and use our content for personal use, or
> use within your organisation. **You must not supply it to any other person** or use it for any
> commercial purpose."

The cyclone database page states no terms (a regex sweep for `creative commons|licen[cs]e|cc.by`
returned zero matches; its only rights link is to that copyright page). Publishing a website
supplies the content to other persons. The two prohibitions are independent, so being free and
non-commercial does not cure it.

IBTrACS is the sanctioned route. NCEI states it holds "agreements from the Regional Specialized
Meteorological Centers that provide track data to make IBTrACS data open for distribution";
BOM is the RSMC for the Australian region, so its analysis reaches the site through the `BOM_*`
columns under citation only. Cross-validated exactly: Gwenda 900 hPa, George 902, Orson 905,
Marcus 905, Yasi 929, Tracy 950 — identical to BOM's own file and to the published records.
All six are pinned as build gates.

**Two corrections to standing notes came out of this:**
1. The internal note attributing BOM's anti-scrape / IP-blocking language to Water Data Online
   is wrong — `bom.gov.au/water/about/copyright.shtml` now 404s and that language sits on BOM's
   **general, site-wide** copyright page. The note understated its scope.
2. Flagged for review, not acted on: `au-warming` relies on an ACORN-SAT CC BY statement scoped
   to *"all maps, graphs and diagrams in this page"*, which is arguably narrower than the data
   files themselves. Worth a look; it is not this run's call.

**Residual risk, stated honestly:** whether NOAA's citation-only grant reaches BOM's underlying
rights in the `BOM_*` values is a genuine open question the NCEI metadata does not address. The
position is still far stronger than sourcing from BOM directly, and bom.gov.au is never touched.

## Architecture Decisions
- **Vanilla TS, not React** — one page, six views, one drill-down.
- **No force graph and no Sankey.** Storms do not connect to each other; a bipartite
  storm↔state graph would look identical for any dataset, which is the failure the brief names.
  Forms were chosen for trajectory data: polylines, small multiples, a coverage matrix, a
  size-vs-strength scatter, an intensification histogram.
- **Two payloads.** Geometry (`tracks.json`) is what the map draws; intensity
  (`profiles.json`, 152 KB gz) is only read inside a dossier and loads on the first drill-down.
  First paint ≈ 414 KB gz instead of 566 KB — the audience is often on a regional connection.
- **Severity by central pressure, never wind.** Gust coverage is 0% before 1970 and ~98% after
  2000; a gust-derived severity chart is a record of instrumentation. Both versions ship side
  by side in the *The record* view so the artefact is visible rather than footnoted.

## Test Results
- Tests written: **174**, across 6 files
- Tests passed: **174**
- Tests failed: 0

Three failures found *real* problems during development and are worth recording:
1. `data.test.ts` asserted the pressure-derived severity series does not rise across all
   decades. It failed — and the **copy was wrong**, not the test. Both series rise early
   (pre-satellite detection), and the true contrast is that their *peaks* fall in different
   decades: gusts peak in the 2010s, pressure peaks in the 1990s and has since fallen to 25.
   The callout was rewritten to say exactly that, plus a third callout admitting the
   pressure chart's early decades are also unreliable.
2. `geo.test.ts` "mid-ocean point" case failed because (-30, 155) is only 165 km off the NSW
   coast — the test was wrong, not the locator. Replaced with genuinely remote points and a
   companion test pinning that a genuinely-coastal point *is* still placed.
3. Two scanners flagged their own explanatory comments. Fixed the scanners; kept the comments.

## Build Status
- npm install: pass
- npm test: pass (174/174)
- npm run build: pass
- Local preview: pass
- Pipeline gates: **21/21 pass**

## Verification
- **Local preview (Chrome MCP + Browser pane):** all six views render; no `NaN`/`undefined`
  anywhere; **no horizontal overflow at 375px on any of the six views** (`scrollWidth ===
  clientWidth === 375` each time).
- **About modal opened over the live Leaflet map** and asserted on computed state: panel
  z-index 2100, scrim 2000, Leaflet's own max z-index 800, `.map-frame` `isolation: isolate`,
  and `elementFromPoint` over the panel returns the panel. The historical "modal renders under
  the map" defect is verified absent.
- **Production:** `https://au-cyclones.benrichardson.dev` → 200; the live HTML references
  `assets/index-CjtG_8e-.js`, byte-identical to the local `dist/index.html`; all data
  endpoints, `og.png`, `robots.txt`, `sitemap.xml` and `third-party-notices.txt` → 200.
  Cert `https_certificate.state = approved`.

### ⚠️ What could NOT be verified
**Real-click interaction testing could not be performed in this environment.** Every click
method — coordinate clicks in both the Browser pane and Chrome MCP, and ref-based clicks —
delivered **zero events** to the page. This was isolated properly rather than assumed: a
capture-phase listener on `document` plus per-event listeners on the target logged nothing for
any click, while `hover` worked (tooltips appeared) and direct handler invocation worked
(`.click()` opened the modal). So it is click *delivery* that is blocked, not the site.

Consequently the four-exit modal dismissal contract is verified by **18 state-asserting jsdom
tests** (including `pointerType: 'touch'` sequences, double-fire guarding, and assertions that
the panel is *removed from the DOM* rather than hidden) and by the computed-style z-index check
above — but **not** by a real click in a real browser. That gap is real and should be closed on
the next run in a session where clicks work.

## Deployment
- Repo created: yes — https://github.com/ben-gy/au-cyclones
- GitHub Pages enabled: yes (workflow build type)
- Cloudflare DNS CNAME: yes — `au-cyclones` → `ben-gy.github.io`
- TLS certificate: approved
- Directory entry live on main: yes
- Workflow triggered: yes — `Deploy to GitHub Pages: success`

## Errors & Resolutions
1. **Data Pipeline failed on its first CI run** — `ENOENT` on
   `/home/runner/work/patterns/geo/au-states.geojson`. The aggregator read the boundary file
   from two directories above the repo root, which only exists inside the factory checkout.
   Fixed: the committed copy in `public/data/` is now the source of truth and the factory path
   is a first-time seed only. Pipeline re-ran: **success**.
2. **Judge's proposed headline was wrong.** It claimed all-storm counts were flat (10.9 → 11.2
   → 11.6) while severe storms declined. Recomputed: **both** decline (14.07 → 12.50 per
   season; severe 5.80 → 3.79). Rewrote the finding, and added the refutation test that makes
   it publishable — pressure coverage *rose* 80.6% → 91.4% across the same period, so the
   decline is measured against an improving record. That check is a build gate.
3. **A fabricated claim in my own copy.** The Records view asserted "Gwenda … never crossed the
   Australian coast." The shipped data says Gwenda *did* cross, into Western Australia. Caught
   because the Coast view's severity table showed Gwenda as WA's deepest crossing. Replaced
   with a computed statement ("11 of these 15 reached land at some point") plus the caveat that
   minimum pressure is not necessarily reached at the crossing.
4. **Browser cache masked four rebuilds.** The pane kept running the *first* build's CSS
   (`index-ClIT7f_Y.css`) while the server had `index-B2KaUbSJ.css`; every measurement taken
   before that was of stale code. Caught by comparing the stylesheet the page actually loaded
   against the one being served — the same check the routine mandates for production.
5. **Map fitted to Australia's coastline** rather than the region, so half of every track ran
   off the frame. Fixed by fitting to the 90°E–160°E region box and sizing the frame by aspect
   ratio rather than viewport height (a `vh` height is fine on a tall display and wrong on a
   short one — at 2.4:1 the "Australian region" view reached the coast of Africa).
6. **Two data-quality defects found and fixed:** `BOM_GUST_PER` is byte-identical to `BOM_GUST`
   on all 11,138 rows where both exist (dropped rather than shipped as a bogus "averaging
   period"); and 66 storms carry dual-agency names joined by a colon (`BERTHA:BERTHE`), now
   normalised to `Bertha-Berthe` to match IBTrACS's own hyphenated spellings.
7. **The provisional-season trap.** IBTrACS publishes the running season's positions long
   before it analyses intensity: at build time 2025-26 had 15 systems, every one `NATURE=NR`
   with null pressure. Charted naively that is a collapse to zero cyclones in the most recent
   season. The pipeline detects the boundary, holds such seasons out of every count and trend,
   and still shows them on the map marked "not yet analysed" so name search still finds them.
8. **Port collisions from concurrent runs** — 5199 and 5241 were both already serving other
   projects from other sessions. Verified the served `<title>` before trusting any preview.

## Per-view UX critique
| View | Question it answers | Hover | Click | Next action |
|---|---|---|---|---|
| Atlas | Has one come past *here*? | Track summary + pressure | Opens the storm dossier | Filter by season/severity, or search a name |
| Seasons | Are there more cyclones than there used to be? | Per-season counts, severe, crossings, measured share | Jumps to the Atlas | Read the two caveat callouts |
| Coast | How exposed is my stretch of coast? | Crossings, severe share, deepest | Drills into every storm behind that jurisdiction | Open a storm; toggle genesis vs landfall |
| Records | What are the extremes? | Exact values per bar | Opens the dossier | Sort the full 1,632-row table |
| Anatomy | Are the strongest storms the biggest? | Radius, pressure, wind, crossing | Opens the dossier | Zoom/pan the scatter |
| The record | Why does the obvious chart lie? | Per-cell coverage, per-decade counts | — (deliberately static) | Compare the two severity charts |

No dead clicks: every `cursor: pointer` affordance is wired. Legend swatches on the Atlas are
real filters, not decoration.
