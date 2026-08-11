# Build Log: Fuel Prices
**Date:** 2026-08-11
**Status:** deployed

## Idea Source
Researched — `IDEAS.md` was empty. Two workflow rounds, 13 agents, all pinned to Opus 5.

**Round 1** (7 probes + 3 judges). Every probe was required to *fetch* its sources and report real
status codes and byte counts. Scores: fuel prices, liquor licences and bulk billing all GO; food
safety GO but NSW-only; complaints GO but no geography; ports GO but stale; ATSB WEAK.
The three judges split three ways — reliability picked fuel, audience picked bulk billing,
visualisation richness picked liquor licences.

**Round 2, adversarial** (3 refutation agents). This is where the run was decided.

### Three findings that changed the outcome rather than confirming it

1. **The headline I was going to ship was refuted, and the truth was better.** The claim
   "Perth petrol is cheapest on Tuesday, a 13–14 c/L cycle that has persisted for years" is wrong
   twice over. The agent computed 57 monthly files across four eras and found the cheap day has
   **moved three times** — no cycle 2005–08, Wednesday 2009–14, **Monday 2015–19 (when Tuesday was
   the single most EXPENSIVE day, by 26.5 c/L)**, Tuesday only from February 2020, datable to the
   month. And "13–14 cents" matches no year of the Tuesday era: the gap was +28c (2020), +48c
   (2024), +39c (2025), +10c (2026). The site ships the regime story instead.
2. **The best-scoring candidate is licence-blocked.** Medicare bulk billing had the strongest
   audience case in the batch. A second agent, briefed only to *refute* the rejection, could not:
   no CC BY on health.gov.au's copyright page, none on the publication pages, and none inside the
   workbooks (it unzipped the xlsx and grepped every XML part). The notice says in terms *"You must
   not reproduce, frame or reformat the files … on any other website"*. Logged to
   `EXPANSION_IDEAS.md` as DO-NOT-BUILD with the full evidence, plus the useful general rule that on
   health.gov.au the licence lives **inside the file**, so a blanket "closed" heuristic would wrongly
   exclude a lot of genuinely open material.
3. **The WA source survived every attack.** The manifest is exactly 308 entries for exactly 308
   calendar months with zero gaps and zero duplicates; its `bytes` field matched real
   Content-Length on both files pulled; CC BY 4.0 confirmed from `package_show`
   (`license_id: cc-by`, `isopen: true`). It also found what the probe missed: **no CORS anywhere**
   (API, blob, or RSS), which rules out a client-side build outright.

This also closes the **Petrol Cycle** item that has sat in `EXPANSION_IDEAS.md` since 2026-08-08 —
its stated objection was that the brand leader/follower finding rested on only two hike waves. It
now rests on 308 monthly files back to 2001.

## Site Details
- **Name:** Fuel Prices
- **Repo:** ben-gy/au-fuel-prices
- **Category:** economic-indicators (retail fuel — a new domain; nothing in 71 sites covered it)
- **Audience:** Perth and Brisbane drivers deciding tonight whether to fill up now or wait, on a
  phone; plus the ACCC/state-treasury analysts and journalists who argue about whether the cycle
  still exists and need a defensible 25-year series
- **Stack:** vanilla-ts + Vite 6, Leaflet 1.9.4 the only runtime dependency, every chart hand-rolled SVG
- **Data strategy:** pipeline, **monthly cron** (day 6, 04:23 UTC). Queensland publishes one file a
  month, which is the slower source and also the fastest cron the factory allows. There is no CORS
  on any endpoint, so a realtime client-side build was never available.

## Data Sources
- **WA FuelWatch historic retail prices** — manifest `fuelwatch.wa.gov.au/api/report/monthly-retail-prices`
  → 308 monthly CSVs on `warsydprdstafuelwatch.blob.core.windows.net`, **2.19 GiB**, 3 Jan 2001 → today.
  CC BY 4.0 per `catalogue.data.wa.gov.au/api/3/action/package_show?id=fuelwatch-historic-fuel-prices`.
- **WA FuelWatch site register** `/api/sites` — 947 stations with coordinates and **tomorrow's price**.
- **Queensland Fuel Price Reporting** — 5 CKAN packages, 91 monthly CSVs, Dec 2018 → Jul 2026, CC BY 4.0.
- **ABS ASGS 2021 SA4** via `geo.abs.gov.au` ArcGIS — 29 real polygons for WA + QLD, 93,837 coordinate values.

**NSW deliberately excluded.** Its bulk history is open and keyless but **CC BY-SA**; mixing
share-alike data into the same derived files would put the whole output under share-alike terms.
Logged as an expansion with the isolation strategy that would make it legal.

## Architecture Decisions
- **Stateless full rebuild, not an incremental cache.** Caching per-month aggregates would commit
  tens of megabytes to the repo to save a download that happens once a month; ~3 GiB parses in
  about 2½ minutes in CI.
- **Per-region series files.** The daily series are 6.5 MB combined — not a payload to hand a phone.
  `series-index.json` carries a weekly-resampled overview so History draws immediately, and each
  region's daily file is fetched only when it is charted.
- **The cycle is identified by the HIKE, not the cheapest day.** See below; this is the single most
  important decision in the build.
- **Two different regionings, deliberately.** WA uses FuelWatch's own ten regions, stable since 2001.
  Queensland has no region column, so its stations are assigned to an ABS SA4 by point-in-polygon
  from the coordinates in every row.

## The method, and the two guards
**The obvious approach is wrong and looks convincing.** A median by day-of-week across a month is
confounded by the price trend: in the worst recent month (March 2026) it hid **73% of Perth's real
cycle**, and across 24 months it captures anywhere between **27% and 269%** of it with nothing in
its own output to say which. Every statistic here is computed one complete Mon–Sun week at a time,
and the cycle is found from the **day-over-day jump** — 20–45 c/L against a drift of about 1 c/L a
day, so it survives any trend. The day before the hike is the day to buy.

**The second guard exists because remote regions genuinely do not move.** An unguarded `argmin`
returns index 0 on every tie, so the site would have told Kimberley drivers to fill up on Monday to
save 0.1 cents — with 86% "reliability" attached. Weeks with a range under 2 c/L are counted and
excluded; 13 regions correctly report no usable cycle at all.

**Both guards are tested where they FAIL.** `tests/cycle.test.ts` asserts that the naive month-wide
method still picks the wrong day, and that the unguarded argmin really does return Monday on flat
weeks. A fixture mistake here was instructive: my first trend fixture applied the drift *per week*,
which shifts every weekday equally and confounds nothing — the guard looked unnecessary until the
drift was applied *per day*, which is how it actually behaves.

## Findings, and what happened to them
**Shipped.** Perth's buy day moved Wednesday (2010–14) → Monday (2016–19) → Tuesday (2020–26),
reproduced independently three times: by the research probe, by the refutation agent on 57 raw
files, and by this pipeline on all 308. In the Monday era Tuesday was the *most expensive* day.

**The two-state contrast is the second finding and needed both.** Perth's cycle is 7.0 days and ran
in 100% of weeks in 2026; Brisbane's is ~36 days (recent ~40) and its weekly trough sits at 0.23–0.33
— close to the 1-in-7 chance level. The site therefore gives a cheap day for Perth and Peel only,
and a cycle *length* for Queensland, rather than handing Perth's answer to a Brisbane driver.

**Deliberately not overstated.** The current Wednesday jump is **10.6 c/L** and the stat card says
"typical Wednesday jump, 2026" — the cycle has attenuated to roughly a third of its 2024 amplitude,
and labelling the year rather than quoting a bigger historical number keeps it honest.

## Traps found in the real data (all gated)
1. **The trailing comma.** WA files carry one up to 09-2025 and not after (11 fields + LF vs 10 +
   CRLF). A `length === 10` assertion silently rejects **297 of 308 files**. The run confirms the
   split as exactly 297/11, and the parser indexes by header name.
2. **Queensland prices are TENTHS of a cent** (1699 = $1.699) and **9999 means "temporarily
   unavailable"**, not $9.99 — 40,184 sentinel rows dropped.
3. **Queensland stamps UTC**, ten hours behind AEST, so evening price changes misfile a day early.
4. **Three Queensland stations report premium fuel at 20.1, 26.9 and 39.9 c/L** — keying errors that
   took over the "cheapest station" list. Filtered by a **relative** test against the same
   product/region/day, never an absolute floor, because unleaded really did sell for 48.7 c/L in
   2001 and any fixed threshold either lets today's errors through or deletes the early history.
5. **January 2001 starts on the 3rd** — the scheme's real start, not missing data.
6. **Product vocabulary drifts** (LRP dies out, E85 and Brand Diesel appear), so every long-run
   series is pinned to ULP, the only fuel present in all 25 years.
7. **`/api/sites` returns one object per site, not an array of products**, and serves ULP however it
   is queried — so tomorrow's price exists for unleaded only, and the site says so.

## Test Results
- Tests written: **163** (25 parser, 19 cycle mathematics, 36 positional layout, 24 against the real
  committed data, 59 source hygiene)
- Tests passed: **163**; failed: 0
- **19 pipeline gates**, all passing, run against the artefacts that ship.
- Failures during development, all real: the two method gates (my own demonstration was wrong, not
  the code — see below); a `DAY_NAMES` short-vs-long mismatch inside a gate; a fixture whose drift
  was applied weekly instead of daily; and **my own explanatory comment tripping my own regex** — the
  workflow says why it does *not* use `npm ci`, so the hygiene test now strips YAML comments first.

### A gate that had to be rewritten because the data disagreed with me
I wrote a gate asserting the naive month-wide method gets June 2026 wrong. It passes on the *hike*
test (Wednesday, 4 of 4 weeks) but the naive method happened to name the right day that month, and
over 24 months it agreed with the correct method **every time**. Rather than force the claim, the
gate now *locates* the failing case programmatically — worst month March 2026, 73% of the cycle
hidden — and asserts the error is unpredictable (27%–269% across months). That is the honest version
and a stronger guard.

## Build Status
- npm install: pass · npm test: pass (163/163) · npm run build: pass · Local preview: pass

## Deployment
- Repo created: yes — https://github.com/ben-gy/au-fuel-prices
- GitHub Pages enabled: yes · TLS certificate: **approved on the second poll**
- Deploy workflow: **success**
- Directory entry live on main: yes
- Pull request: none, per the routine

### DNS — the zone is still at its cap
`benrichardson.dev` remains at **200/200 records**, the free-plan limit. Rather than fail, I
repointed one of the four confirmed-dead records left by the 2026-08-04 batch:
`au-build-approvals` → `au-fuel-prices` (record `df431b2e8197f6dea1fc3167964b798e`). It was verified
dead first — no matching repo, no TLS certificate ever issued, `https=000`.
**Three dead records remain, so the next three runs are covered, then this blocks again.** The
durable fix is a paid Cloudflare plan or a deliberate prune, which is the user's call.

## Verification
**Live production URL, by HTTP:** `/`, `data/meta.json`, `data/cycle.json`, `data/sites.json`,
`data/sa4.geojson`, `data/series/WA-metro.json`, `og.png`, `robots.txt`, `sitemap.xml`,
`favicon.svg`, `third-party-notices.txt` and the IndexNow key all return **200**. The live bundle
(`index-g6-Jdx3O.js`) **matches** the local `dist` exactly.

**Browser verification ran against the LIVE production URL.** All eight views render with content,
charts and tooltips; **zero console errors**; no `NaN`/`Infinity`/`undefined` in any SVG attribute or
any visible text; zero native `<title>` elements used as tooltips.

**The full dismissal contract passes**, asserted by computed style and re-opened between each exit,
**from the map view** (where Leaflet could paint over it): the ✕, the scrim by mouse `pointerdown`,
the scrim by a real `pointerType:'touch'` sequence with no double-fire reopen, and Escape — plus a
clean baseline (no `[hidden]` element rendering, `elementFromPoint` at the viewport centre returning
page content) and the scroll lock released. Modal z-index 2100; `.map-frame` carries
`isolation: isolate`.

**Real trusted clicks on production**: a station marker opened the drawer for Solo Hamilton Hill
(Metro WA) with five fuels and **tomorrow's unleaded price, 189.5c, −1.8c** — the WA-only feature
working end to end. A suburb row opened the Beckenham drill-down.

**No page-level horizontal overflow at 375px on any of the eight views**, nor with the drawer or the
About modal open — asserted by attempting `window.scrollTo(200,0)` and reading `scrollX` back as
**0**, not by eyeballing a screenshot. Tables and charts scroll locally inside their own containers.

### Four defects were found by looking, and all four were fixed
1. **The regime ribbon used a sequential price ramp for a diverging quantity.** Each cell is a day's
   distance from its own week's average, but on a six-step cheap→dear ramp the midpoint lands on
   orange — so "exactly average" read as expensive and the no-cycle years 2001–08 painted as a wall
   of orange. Replaced with a proper diverging scale centred on neutral; the story only became
   legible after this.
2. **"Today" was ranking a two-month-old price first.** The cheapest station was a truck stop that
   last reported on 11 June. Stations are now excluded from Today unless they reported within a
   fortnight of that state's latest data, and the 37 excluded are stated rather than hidden.
3. **The tank-difference stat was set by a single remote roadhouse** — "$155.16 a tank" is true and
   useless. Now the 10th-to-90th percentile: **$20.35**, the money actually on the table.
4. **The ribbon and the Sankey hugged the left edge on desktop**, sized to a fixed 588px and 880px
   inside ~1300px panels. Both now size to their container and still scroll locally on mobile.

### A measurement trap that produces confident wrong readings
The in-app browser pane reported `innerWidth: 0` while backgrounded and returned **composited stale
screenshots** — one frame showed the header painted halfway down the page over content from a
different scroll position. Nothing was wrong with the site; the DOM measurements were consistent
throughout. Verification was redone in real Chrome. Separately, Chrome's screenshots are scaled
**1456/1728 = 0.843** from CSS pixels, so a click computed from `getBoundingClientRect()` lands in
the wrong place unless it is converted — the first two attempts to click a station "did nothing"
for exactly this reason, and the marker was fine.

## Errors & Resolutions
- **The Data Pipeline failed in CI on its first run** — and this is the most valuable thing the run
  found. `data.qld.gov.au` served the GitHub runner an **HTML interstitial with HTTP 200**, so
  nothing retried and the failure surfaced much later as a confusing "QLD CSV missing required
  columns". Two fixes: request files as a **navigation** (`Sec-Fetch-Dest: document`,
  `Mode: navigate`, `Site: none`) rather than as a cross-origin XHR, which is a combination no real
  browser sends; and **validate the body, never the status code** — `fetchText` now takes an
  `expect` predicate and retries when a CSV request returns something starting with `<`.
  `dns.setDefaultResultOrder('ipv4first')` added for the same class of CI-only failure.
- `/api/sites` returns `product` as a single object, not an array — the first run crashed on it.
- The `regionSlug` used by the frontend must match the pipeline's byte for byte, or every series
  file 404s; a test asserts every indexed region has a file whose arrays match the date axis.

## Honest limitations, all stated on the site
- **Two states only** — about a third of the population. NSW is excluded on licence grounds, not
  technical ones, and the reason is given in the About panel.
- **No cheapest-day advice outside Perth and Peel.** Brisbane's cycle is ~36 days; remote regions
  barely move. Both are said plainly rather than given a precise-looking meaningless number.
- **Prices are nominal** — 2001 figures are not inflation-adjusted.
- **Queensland figures are carried forward** from the last reported change, never more than 60 days.
- **286 stations have no published coordinates** and are absent from the map but present in every
  table — stated in the map's own caption.
- **Tomorrow's price is unleaded only**, because the register serves only unleaded.

## Licensing
AGPL-3.0-or-later with the section 7(b) attribution term; `LICENSE` (verbatim, fetched from the
GitHub licences API), `ADDITIONAL-TERMS.md`, `CONTRIBUTING.md`, `THIRD-PARTY-NOTICES.md` +
`public/third-party-notices.txt`, SPDX headers on every first-party file under `src/` and
`pipeline/`, and `"license": "AGPL-3.0-or-later"` in `package.json`. The only third-party runtime
component is **Leaflet 1.9.4 (BSD-2-Clause)**, derived from the production sourcemap's `sources`
rather than from `package.json`. No third-party fonts. A gate asserts the WA source resolves to
`cc-by` and not share-alike, and `ADDITIONAL-TERMS.md` states explicitly that the data licences are
separate and not mine to change.
