# Build Log: Air Episodes
**Date:** 2026-08-25
**Status:** deployed

## Idea Source
IDEAS.md was empty. The idea came from `EXPANSION_IDEAS.md`, which carried a verified, explicitly
flagged entry from the 2026-08-19 research round:

> **NEW SITE (not an expansion) — Ambient air quality**: hourly *measured* pollutant
> concentrations at government monitoring stations, which is a different thing from
> **au-pollution** (the National Pollutant Inventory: annual, self-reported *estimates* of
> emissions by industrial facilities).

That entry named three obtainable jurisdictions (NSW, QLD, ACT) and rejected Victoria outright.
An independent verification round this build **overturned the Victoria rejection on the facts and
then re-rejected it on a different, stronger ground** — see below.

## Site Details
- **Name:** Air Episodes
- **Repo:** ben-gy/au-air-episodes
- **Category:** environment / data-explorers
- **Audience:** people who want to know what actually happened in the air where they live —
  smoke-season residents, asthma and COPD households, parents, local journalists, council
  environmental-health officers. Mobile-first, often at night, often *during* an event.
- **Stack:** Vanilla TypeScript + Vite 6, Leaflet 1.9 from npm, hand-rolled SVG for every chart
- **Data strategy:** hybrid — a **monthly** Node pipeline (`23 4 9 * *`) for the archive, plus a
  **client-side realtime layer with no scheduled Action at all**. Cadence justification: the
  slowest input is annual (Queensland's validated CSVs land about mid-year), the fastest
  committed input is NSW's rolling hourly, and monthly is the factory floor. Anything faster
  would re-download an archive that changes once a year.

## Data Sources
| Source | URL | Licence |
|---|---|---|
| NSW Air Quality Data Services | https://data.airquality.nsw.gov.au/api/Data/get_Observations | CC BY 4.0 (SEED record f2abe03c) |
| QLD Air Quality Monitoring (grouped by pollutant) | https://www.data.qld.gov.au/dataset/air-quality-monitoring-{year}-grouped-by-pollutant | CC BY 4.0 |
| QLD live measurements | https://airquality.des.qld.gov.au/v1/stations/parameters/measurements | CC BY 4.0 |
| ACT Air Quality Monitoring | https://www.data.act.gov.au/resource/94a5-zqnn.json | CC BY 4.0 |
| ABS ASGS state boundaries | `patterns/geo/au-states.geojson` | CC BY 4.0 |

**20,033,621 hourly readings** across **98 station clusters** (from 271 source stations),
2009 → now for Queensland, 2011 → now for the ACT, 2016 → now for NSW hourly and 1994 → now for
NSW daily.

## Research (ultracode, Opus 5 throughout)
An 8-agent Workflow ran before any code: three independent source verifiers (one per
jurisdiction plus a national sweep), one standards researcher, three competing design proposals,
and a judge that re-tested the four claims the proposals disagreed on. 1.04M subagent tokens,
360 tool calls. Every agent pinned to `model: 'opus'`.

**It overturned two of the prior round's conclusions:**
- **Victoria is NOT technically unavailable.** `discover.data.vic.gov.au` publishes EPA AirWatch
  hourly workbooks — open, keyless, 16 stations including Melbourne CBD, 884,554 rows for 2025
  alone. The CKAN record tags them `cc-by`.
- **South Australia does NOT stop in 2020.** That was a wrong-base-path artefact: the CKAN base
  is `data.sa.gov.au/data/api/3/...`, not `data.sa.gov.au/api/3/...`. SA is live to July 2026.

**Both were still excluded, for better reasons** (see "Jurisdictions excluded").

## Architecture Decisions
- **Vanilla TS, no framework.** Eight hash-routed views over static JSON; nothing here needs a
  component tree.
- **Hand-rolled SVG for every chart.** Lorenz small multiples, a dual-rank slope table, a
  station × hour matrix, an hour × month calendar, a polar pollution rose and a three-scale
  comparison strip. No charting library was added; none of these is a library default.
- **Deliberately NOT used:** a force-directed network (there are no entity-to-entity edges in
  this data) and a treemap (nothing is hierarchical). Reaching for the house defaults would have
  been wrong for a high-frequency multi-station time series.
- **Dark-first**, overriding the factory's "environment leans light" default, because the
  concentration ramp has to *darken* at its top end — a ramp running to bright yellow makes the
  worst hour ever recorded the lightest mark on the page and inverts the message.
- **Hybrid data strategy.** NSW cannot be live (verified: no `Access-Control-Allow-Origin` header
  at all, and its OPTIONS preflight 404s), so the live map covers Queensland and the ACT only and
  says so on the map rather than hiding the asymmetry.

## The headline
> At the typical monitoring station, the worst 1% of hours in a year — about 88 of them — carry
> **5.8%** of that year's total PM2.5 exposure.

Median of **669 station-years**, IQR 4.8%–7.9%. Per jurisdiction: NSW 5.8%, QLD 5.6%, ACT 6.8%.
Recomputed at a 95% data-capture threshold: 5.8% (Δ 0.0 pp). Recomputed excluding the Black
Summer years 2019–20: 5.5% (Δ 0.3 pp) — so it is not a bushfire artefact. It is chosen because
it is *undefined on the daily and annual files every official report is built on*; only the
hourly record can answer it. The site also names the extreme: the most burst-dominated
station-year in the archive delivers **46%** of its exposure in 88 hours.

## Jurisdictions excluded, and why (this is a finding, not an apology)
- **Victoria — licence conflict.** The CKAN record says CC BY 4.0; the XLSX the user actually
  downloads embeds an *EPA Victoria Terms of Use* granting rights on a **non-commercial** basis.
  CC BY 4.0 expressly permits commercial use, so the two cannot both be true, and the restrictive
  term is the one embedded in the artefact. Same shape as the ACARA block already on record.
- **Tasmania — ShareAlike contagion.** BLANkET is CC BY-**SA** 4.0; folding it in risks forcing
  ShareAlike onto the whole derived work. Also snapshot-only and explicitly not quality
  controlled.
- **South Australia — insufficient verification.** Genuinely open and current, but exactly one of
  its ~20 station archives was downloaded and opened. Shipping the rest on catalogue metadata
  would be asserting something never checked. Logged to EXPANSION_IDEAS.md as the top v2 item.
- **WA** publishes no open ambient series; **NT** has one 2021 catalogue record pointing at a web
  application.

## Test Results
- Tests written: **145** across 5 files (parse, format, layout, overlay, hygiene)
- Tests passed: **145** — Tests failed: **0**
- Build-time reconciliation gates: **31 of 31 pass**

Three failures were found and fixed during the run rather than shipped:
1. `PointerEvent` is not implemented in jsdom, so the touch-exit overlay tests could not run.
   Shimmed rather than downgraded to `MouseEvent` — the touch exit is the one that breaks in the
   field, so testing it with a mouse event would have tested nothing.
2. A test asserting `LICENSE` does not contain `MIT` false-failed on PERMITTED/LIMITED/TRANSMIT
   inside the AGPL text.
3. `.overlay-body` used `overflow-x: hidden`; changed to `clip` so the blanket sticky rule holds.

## Defects the gates and the visual pass caught (all fixed before deploy)
1. **Queensland ingested ZERO readings, silently.** The grouped CSV headers name a station
   `Barney Point (Gladstone Region, Central Queensland) (ug/m^3)` while the station registry
   calls it `Mobile: Barney Pt`. Every file parsed, every row count was right, and every column
   was discarded. Every other gate passed. Fixed with a two-stage strip plus a small set of
   unambiguous aliases (99 of 111 columns now match), and a **new gate** that fails the build if
   the join rate drops.
2. **A null-island cluster.** Nine Queensland stations publish a latitude and longitude of
   exactly `0` — a sentinel for "no location released". Left in, all nine sat within 2 km of each
   other in the Gulf of Guinea, so the clustering merged **nine unrelated stations into one**,
   put a dot off the coast of Africa on the map, made it the site's default station, and made
   every episode containing it appear to span **15,936 km**. Fixed with an Australian
   bounding-box test, a gate, and a Record-view panel naming the nine.
3. **Episodes merged across the continent.** The first detection rule cut a single national
   timeline into runs, so a smoke event in Cairns and an unrelated one in Wagga on the same night
   became one "episode" — and because something somewhere in a 99-station network is nearly
   always elevated, those merges chained into a single run lasting **64 days**. Replaced with
   spatio-temporal connected components: two elevated cluster-hours join only if they are close
   in time **and** within 250 km. Widest episode is now 860 km, longest 27 days, and both are
   gated.
4. **The propagation matrix drew 94,491 SVG cells** for the longest episode and froze the
   renderer. Now windowed to the 24 highest-peak clusters and ±84 hours around the peak, with an
   explicit notice of what is hidden and a "draw all of it" control.
5. **Wind vectors stole clicks from station markers.** A Leaflet polyline is interactive by
   default and is drawn from the marker's own centre, so it swallowed the click meant to open the
   station. Found by a real trusted click, fixed with `interactive: false`.
6. **The segmented control never updated its own selected state** — it set classes at build time
   only, so after any click it showed the wrong segment while the view changed underneath it.
7. **The pollution rose was monochrome.** On the published absolute scale, every wedge at a
   coastal city falls in the "Good" band — true, and useless. Added a default "Compared with
   here" scale (fixed ratio breaks against the station's own median hour, in a separate cold-blue
   hue family with its own legend), keeping both absolute scales one click away.
8. **Leaflet's stylesheet overrode the map background**, putting a light-grey slab behind a dark
   map wherever a tile had not yet arrived. Also made the basemap follow the reader's theme.

## Build Status
- npm install: pass
- npm test: pass (145/145)
- npm run build: pass — 245 kB JS (78 kB gzipped), 34.5 kB CSS
- Local preview: pass
- Production verification: pass

## Verification performed
- All 8 views rendered and inspected by screenshot at 1440px and asserted at **375px**: no
  horizontal overflow on any view (`scrollWidth === clientWidth` on all eight).
- **The modal dismissal contract**, tested by *state* on the live page, with the About modal
  opened **from the map view** (where Leaflet's panes would paint over it): panel z-index 2100 vs
  Leaflet's maximum 800, `elementFromPoint` at the viewport centre inside the panel. All four
  exits verified independently with re-opening between each — ✕, scrim pointerdown, Escape, and a
  real `pointerType: 'touch'` sequence — plus the touch-double-fire case. All pass; no leaked
  `[hidden]` elements.
- **A real trusted click** on a map marker opened the Wagga Wagga drill-down: 5,212 days
  measured, 50 days over the Australian standard, 352 over the WHO guideline, worst day
  560 µg/m³, with a full daily chart.
- Live production URL returns 200 and serves the **identical bundle hash** as the local dist.
  `og.png`, `robots.txt`, `data/meta.json` and `data/episodes.json` all 200.
- **Zero console errors** on the production site.

## Deployment
- Repo created: yes — https://github.com/ben-gy/au-air-episodes
- GitHub Pages enabled: yes
- Cloudflare DNS CNAME created: yes
- TLS certificate: `https_certificate.state = approved`
- Workflow triggered: yes — first run failed with `HttpError: Not Found` because it raced the
  Pages enablement; re-dispatched and succeeded.
- Directory entry live on main: yes

## Errors & Resolutions
- **The `Data Pipeline` workflow auto-triggered on the initial push** (its `paths: ['pipeline/**']`
  trigger) and would have re-downloaded several hundred megabytes in CI for data already
  committed. Cancelled. Worth noting for future builds: that trigger fires on the very first push.
- **The first deploy failed** on a race with Pages enablement (404 from the deployment API). The
  build had succeeded and the artifact existed; a re-dispatch of `deploy.yml` was enough.
- The in-app browser pane repeatedly reported a zero-width viewport and throttled timers while
  hidden, which made a first overflow reading look like a failure. Verified through both browser
  surfaces before believing either.
