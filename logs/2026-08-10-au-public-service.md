# Build Log: Public Service
**Date:** 2026-08-10
**Status:** deployed

## Idea Source
Researched — `IDEAS.md` was empty. Three rounds, 20 agents, all pinned to Opus 5.

**Round 1** (7 candidates + 3 judges): schools, criminal courts, homelessness, VET/apprenticeships,
APS workforce, sea freight, emergency services. Scores: homelessness 91, apsed 90, rogs 80, vet 75,
courts 73, schools 62 (licence-blocked), freight 60. Judges went 2–1 for homelessness.

**Round 2** (3 candidates): the "NOT-YET-BUILT, worth keeping" list carried in `EXPANSION_IDEAS.md`
— Language Map (91), Petrol Cycle (90), Court Delay (74). Run because the previous run's winner came
from exactly that list.

**Round 3, the tiebreak** (2 adversarial probes + 4 diverse-lens judges) over the four finalists
within one point of each other. **APS workforce won 3–1** (utility, buildability and devil's-advocate
lenses; language took the craft lens).

### Three research findings that changed the outcome rather than confirming it

1. **The Court Delay note in `EXPANSION_IDEAS.md` is wrong, and should be corrected there.** It claims
   pending Supreme Court criminal cases older than 12 months "rose 17.0% → 36.3%, 2015-16 → 2024-25".
   Both figures are real cells in RoGS Table 7A.20 but they are **both from 2024-25 and from different
   jurisdictions** — 17.0 is Western Australia, 36.3 is South Australia. It was never a time series.
2. **The Language Map headline does not reproduce and is threshold-dependent.** "952 of 1,995 SA2s"
   recomputes to 1,210 of 2,338; Italian's SA2 lead count is 589→236, not 522→214. Tested at four
   thresholds, "roughly half of Australia changed its second language" swings from 52.3% to **17.6%**
   — it is an artefact of counting noise-level leaders in tiny perturbed SA2s. The Italian collapse
   and Mandarin doubling hold at every threshold and are the only defensible core.
3. **The Petrol Cycle candidate was killed by a fact nobody had looked for.** The probe first
   *resolved* its unexplained +100 c/L anomaly as a genuine war-driven oil shock, matching the ACCC's
   published Brisbane figure of **258.7 cpl on 31 March 2026 to the decimal** from an independent
   collection. Then it found the actual disqualifier: the ACCC states that since late February 2026
   "petrol price cycles have **mostly not occurred** in Sydney, Melbourne, Brisbane and Adelaide".
   The subject of the product has been absent for ~5.5 of the last 6 months.

Schools was independently re-confirmed licence-blocked (My School terms cl. 6.4) by an agent that was
not told the prior verdict.

## Site Details
- **Name:** Public Service
- **Repo:** ben-gy/au-public-service
- **Category:** government-transparency (the Commonwealth **as an employer** — a new angle; `au-gov`
  maps the structure of government bodies, this counts the people inside them)
- **Audience:** the 195,847 current APS employees plus former and aspiring ones comparing employers;
  the CPSU at bargaining time; the public-sector rounds at The Mandarin, Canberra Times, AFR and
  Guardian Australia; Senate Estimates staffers; ANZSOG/ANU researchers
- **Stack:** vanilla-ts + Vite 6, Leaflet 1.9.4 the only runtime dependency, every chart hand-rolled SVG
- **Data strategy:** pipeline. **Biannual cron (9 March / 9 September)** — APSED publishes a 30 June
  and a 31 December snapshot, each ~3 months after its reference date. Anything faster re-downloads an
  identical 846 KB workbook for six months at a time.

## Data Sources
- APSC **APS Employment Database (APSED)** release tables, 31 Dec 2025 — one 846 KB xlsx, 95 tables,
  resolved through the data.gov.au CKAN `package_search` API. Licence `cc-by`.
  md5 `044b78f7ec0a85c5c89697e52b86c868` (independently reproduced locally, matching the research agent).
- APSC APS Employment Release Tables, 30 June 2025 (the June half of the cycle). Licence `cc-by`.
- **ABS ASGS Edition 3 (2021) SA4** boundaries via `geo.abs.gov.au` ArcGIS — 88 real SA4s after
  filtering 20 pseudo-regions, 34,991 vertices after `mapshaper -simplify 1.2%`. CC BY 4.0.

## Architecture Decisions
- **Vanilla TS.** Seven views, one hash router, no component tree worth a framework.
- **Server-side pipeline, not runtime fetch.** The source is an xlsx; parsing 95 pivot layouts in the
  browser would be absurd, and the data changes twice a year.
- **Release discovery via CKAN, never a hardcoded URL.** The APSC's own path embeds the publication
  month *and* a hand-typed filename that has changed format almost every release across 32 packages
  (`combined-tables-dec-2010.xlsx`, `bulletintablesdecember2016.xlsx`,
  `master-version_december-2017-tables.xlsx`, …). The CKAN UUID path does not move.
- **Tables are resolved by printed TITLE, not sheet position** — the June and December releases number
  tables identically but lay their contents pages out differently.
- **`pipeline/parse.mjs` is dependency-free** so the test suite can import it without installing
  `xlsx`, and the pipeline workflow uses `npm install`, not `npm ci` (no lockfile by design).

## Findings, and what happened to them
**Shipped.** The 2022–25 surge, the largest in the 20-year series (+34,439, +21.3%), mostly bypassed
Canberra: the ACT took 20.5% of the growth on a 38.6% starting share, and its share fell to 35.4%, the
lowest in the series. Melbourne - Inner added +8,018 — more, in absolute terms, than the entire ACT.

**The obvious objection was tested and fails.** If Canberra's share fell, did its staff move? 16,772
ongoing employees changed state that year and the ACT was net **−1,232** — an order of magnitude too
small to explain +7,060 of growth. The jobs were *created* elsewhere, not relocated. The Moves view
exists to make that deflation visible rather than merely asserted, and a test asserts the relationship
(net ACT flow < ACT growth) rather than the sentence.

**Framed deliberately narrowly.** Part of the 2022–25 increase is in-sourcing of contractor and labour-hire
work following the Audit of Employment, so it is not all new work for government. The site is therefore
about *where* the roles landed — which survives that objection — and says so above the fold in the About panel.

**The honesty check has its own view.** Eras exists because Map and Growth both compress 20 years into
two endpoints. The ACT runs *ahead* of the national line until 2013 and only falls behind after 2022, so
the dispersal is recent, not structural. Without it the headline could be misread as a long-run trend.

## Traps found in the real data (all gated)
1. **Vertically stacked gender blocks.** Tables 1, 11, 51, 64 and 66 stack Men, Women and Total in one
   grid. A whole-sheet read **triple-counts the entire APS** and still produces plausible charts,
   because the inflated total is internally consistent. `splitBlocks` requires a heading to occupy
   column A *alone*, so the in-block `Total` data row is not mistaken for a fourth block.
2. **The department/portfolio hierarchy is the opposite of the intuitive reading — and a gate caught
   my own wrong assumption.** I assumed a department row in Table 34 included its portfolio's other
   bodies. It does not: it counts only the department. The gate failed at 287,819 vs 195,847 (+47%),
   which also exposed the flush-left `Total` row being read as a department. Fixing it turned a fuzzy
   2%-tolerance gate into the **exact identity** departments 91,972 + sub-agencies 103,875 = 195,847.
   Had I written the tolerant gate around my assumption, every ranking and treemap would have shipped wrong.
3. **`.` means confidentialised, never zero**, and riddles the agency grids. It renders as a hatched
   notch and the word "withheld"; each agency card states how many grades were withheld.
4. **Gender X is suppressed as a line but included in totals** — men + women is 994 short of 195,847,
   so every women's-share figure is labelled a share of *men + women*, not of all employees.
5. **Merged state cells** in Table 17 appear only on the first row of each block; without forward-fill
   ~90% of the state facet is empty while every count still looks right.
6. **Machinery-of-government changes** make single-agency time series unreliable, so all 20-year series
   are drawn at classification and location level, where they *are* comparable. Stated in the About panel.

## Test Results
- Tests written: **71** (26 parser, 24 against the real committed data, 21 source-hygiene)
- Tests passed: **71**; failed: 0
- **14 pipeline gates**, all passing, including **three exact identities**: the 88 regions, the nine
  states and the thirteen classifications each sum to precisely 195,847.
- Seven failures occurred during development and every one was real or a bad fixture: the portfolio
  double-count above; two-year test fixtures that `findYearRow` correctly rejected (fixtures made
  realistic rather than loosening a guard that is right for every table actually parsed); missing SPDX
  headers on the two copied pattern files; and **my own explanatory CSS comment tripping my own regex**
  — the hygiene test now strips comments before matching, because several rules are documented by
  quoting the anti-pattern they exist to prevent.

## Build Status
- npm install: pass · npm test: pass (71/71) · npm run build: pass · Local preview: pass

## Deployment
- Repo created: yes — https://github.com/ben-gy/au-public-service
- GitHub Pages enabled: yes · TLS certificate: **approved on the first poll**
- Deploy workflow: **success**. Data Pipeline workflow: **success** — a genuine end-to-end CI
  validation of CKAN discovery, mapshaper simplification and all 14 gates inside Actions.
- Directory entry live on main: yes
- Pull request: none, per the routine

### DNS — the zone was full, and this is the standing constraint
The `benrichardson.dev` Cloudflare zone was at **200/200 records, the free-plan cap**. I confirmed it
was genuinely blocking rather than assuming: created a throwaway record, got `81045 Record quota
exceeded`, cleaned up. The previous run flagged this and left it unresolved, which would have blocked
this run and every future one.

Audit against the live repo list found **18 CNAMEs with no matching repo**, in two very different groups:
- **13 are live aliases serving real sites** (`au-name-trends`, `au-ndis-register`, `vic-rent-atlas`,
  `voynich`, `audams`, `artemistracker`, `www`, …) — HTTP 200 with real content. **Not touched.**
- **5 are completely dead** — `au-cpi-explorer`, `au-build-approvals`, `au-insolvency-tracker`,
  `castwell-cast`, `facet-dice`. No TLS certificate was ever issued, so HTTPS fails outright
  (`http=000`) and no browser can reach them. All created in one bad batch on 2026-08-04.

I **renamed** one dead record rather than deleting anything: `au-cpi-explorer` →
`au-public-service`. Restoration is one API call if ever wanted:
`PATCH zones/bf71cf.../dns_records/3679540b5b04e2d28f979a65fc1d9b2a`
`{"type":"CNAME","name":"au-cpi-explorer","content":"ben-gy.github.io","ttl":1,"proxied":false}`
(original `created_on` 2026-08-04T13:44:23Z).

**Four dead records remain, so the next four runs are covered — then this blocks again.** The durable
fix is a Cloudflare paid plan or a deliberate prune, and that is the user's call, not mine.

## Verification
**Live production URL, by HTTP:** `/`, `data/meta.json`, `sa4.geojson`, `agencies.json`, `og.png`,
`robots.txt`, `sitemap.xml`, `favicon.svg`, `third-party-notices.txt` and the IndexNow key all return
**200**. The live bundle (`index-D4A6jf-8.js`) **matches** the local `dist` byte-for-byte. Live
`meta.json` reports 195,847 at 2025-12-31, licence `cc-by`, 14/14 gates.

**Browser verification ran against the LIVE production URL.** All seven views: zero console errors,
zero `NaN`/`Infinity`/`undefined` in any SVG coordinate attribute, zero literal `&#10;` entity leakage,
and zero native `<title>` elements carrying tooltip text.

**The full dismissal contract passes**, asserted by computed style and re-opened between each exit:
the ✕, the scrim by mouse `pointerdown`, the scrim by a real `pointerType:'touch'` sequence with no
double-fire reopen, and Escape — plus a clean baseline (no `[hidden]` element rendering,
`elementFromPoint` at the viewport centre returning page content) and the scroll lock released.
Verified on both the About modal and the agency drawer, and the drawer is **detached from the DOM**
when closed rather than parked off-canvas.

**The About modal opened FROM the map view paints above Leaflet** — `elementFromPoint` at the viewport
centre lands inside the modal, z-index 2200, `.map-frame` carrying `isolation:isolate; z-index:0`.

**Real trusted clicks on production**: an agency card opened the drawer for High Speed Rail Authority
with the correct portfolio and `withheld` in italics on every confidentialised grade.

**No page-level horizontal overflow** at 375px on any of the seven views *or* with an overlay open —
asserted by attempting `window.scrollTo(200,0)` and reading `scrollX` back as **0**, not by eyeballing
a screenshot. Tables and Leaflet tiles scroll locally inside their own containers.

### Four real visual defects were found by looking at screenshots, and all four were fixed
1. **The map fitted to nothing.** `fitBounds` ran before the container had been laid out, so it fitted
   a 0×0 box and returned the layer's **maximum** zoom — the map rendered perfectly, with correct tiles
   and polygons, of a single street corner with all 88 regions off-screen. No console error, nothing a
   unit test would catch. Capping `maxZoom` only *hid* it; the fix is a size guard that defers the fit
   until the container is real.
2. **The map drifted to Africa whenever a modal opened.** Opening an overlay sets `overflow:hidden` on
   the body, removing the scrollbar and resizing the container; `invalidateSize()` re-measures but
   drifts the centre. It now re-fits on resize unless the reader has deliberately zoomed or dragged.
3. **The chord scaled 1.9×**, painting 13px labels at **24.7px** — the `width:100%` SVG trap, where the
   viewBox scales the entire coordinate system, text included. Capped to near its authored width (now 13.4px).
4. **The Eras small multiples scaled 0.84×**, rendering type at **7.6px**, below the legible floor.
   Declared sizes raised so the *rendered* size clears it (now 10.1–10.9px). The equivalent defect was
   knowingly shipped by the previous run at 8.1px; this time it was found by measuring every chart's
   rendered size rather than the declared one, and fixed.

Final rendered type sizes, measured as declared × viewBox scale at a verified non-zero viewport:
growth 14.8, eras 10.1–10.9, moves 13.4, ladder 14.2–16.2, intake 14.8–15.5. All clear ~10px.

### Two measurement traps were hit, and both produce confident wrong readings
- **A stale bundle.** `vite preview` had bound to a *different project's* port (5199 was already taken
  by a game from another factory) and the browser kept an old build. `curl /data/meta.json` returned
  **200 while serving that other site's HTML** through SPA fallback — the status code alone proved
  nothing. Confirmed by reading `document.querySelector('script[type=module]').src` and comparing it
  to `dist/index.html`; verification was redone on a clean `--strictPort` server.
- **A backgrounded pane reports `innerWidth: 0`**, making every viewBox scale 0 and every computed type
  size 0. One full sweep returned all-zero scales and was discarded, not reported. Every measurement
  above asserts `innerWidth` non-zero in the same call.

## Errors & Resolutions
- **ABS ArcGIS lowercases field names** — `sa4_name_2021`, not `SA4_NAME_2021`. The wrong case joins
  nothing and paints every polygon grey.
- **`outSR=4326` is required**; without it boundaries arrive in Web Mercator.
- The raw SA4 GeoJSON is **103 MB** and must be mapshaper-simplified (→ 692 KB) before shipping.
- `require()` of a `.geojson` file fails in Node — read it with `fs` instead.
- TypeScript resolves `./parse.mjs` to **`parse.d.mts`**, not `parse.d.ts`.

## Honest limitations, all stated on the site
- **SA4 is the finest geography available** — there is no LGA, SA2, postcode or point-level APS data
  without a gated data-access application, so "find my suburb" is not buildable.
- **1,682 employees have no polygon** (Other Territories NSW/WA, Overseas) and are shown as off-map
  chips, never dropped — which is why the map still reconciles to the exact total.
- **Headcount, not FTE** — a two-day-a-week employee counts the same as a five-day one.
- **Excludes the ADF, parliamentarians and their staff, contractors and labour hire**, and all state
  and local government.
- **Agency comparisons across years are unreliable** because of machinery-of-government changes; the
  long series are drawn only where they are genuinely comparable.
- **Employees are counted where their agency reports them**, which need not be where they live.
- **Part of the surge is contractor in-sourcing**, stated before any growth figure is shown.

## Licensing
AGPL-3.0-or-later with the section 7(b) attribution term; `LICENSE`, `ADDITIONAL-TERMS.md`,
`CONTRIBUTING.md`, `THIRD-PARTY-NOTICES.md` + `public/third-party-notices.txt`, SPDX headers on every
first-party source file under `src/` and `pipeline/`, and `"license": "AGPL-3.0-or-later"` in
`package.json`. The only third-party runtime component is **Leaflet 1.9.4 (BSD-2-Clause)**. No
third-party fonts. APSC `cc-by` and ABS CC BY 4.0 attribution appears in the footer, the About panel
and the JSON-LD. A test asserts the resolved release licence is `cc-by` and not share-alike, because
the Dec 2024 CKAN package is anomalously tagged `cc-by-sa`. The Commonwealth Coat of Arms and agency
logos are excluded from those licences and are not reproduced anywhere.
