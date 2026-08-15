# Build Log: Radio Spectrum
**Date:** 2026-08-15
**Status:** deployed

## Idea Source
`IDEAS.md` was empty. The idea came from `EXPANSION_IDEAS.md`, where the 2026-08-13
round had recorded **au-spectrum** as a strong runner-up: *"deferred 2026-08-13
(runner-up), build it next time with the framing corrected."* That note carried four
corrections which were all honoured:

1. **The pitched headline was a fleet duplicate.** au-towers already ships a frequency
   ladder for Telstra/Optus/TPG. This site leads instead with *who holds the spectrum* —
   the 99.75% of the register that is not a mobile carrier — and links to au-towers rather
   than competing with it.
2. **Natural-person privacy is a hard licence condition**, not a nicety.
3. **The quoted-comma trap is real** (12,273 of 129,334 site names).
4. **There is no history in the file** — it is a daily snapshot.

Correction to the inherited research: it said there was no long time axis at all. There is
one — `device_details.AUTHORISATION_DATE` survives licence renewal (43.6% of licences hold a
device authorised before the licence's own issue date) and reaches 1959. It is a
survivorship curve, not a registration series, and is drawn as one.

## Site Details
- **Name:** Radio Spectrum
- **Repo:** ben-gy/au-spectrum
- **Category:** government-transparency
- **Audience:** RF engineers, spectrum managers and amateur operators who will spot a wrong
  number instantly; plus journalists and residents asking who is licensed to transmit where
  they live
- **Stack:** Vanilla TypeScript + Vite 6 + Leaflet 1.9, no framework
- **Data strategy:** pipeline, monthly cron (`23 17 6 * *`). The ACMA rebuilds the register
  daily, but this site answers structural questions that move on the scale of auctions and
  machinery-of-government changes; monthly is also the fastest the factory allows for a
  non-realtime explorer, and each run pulls 60 MB from the ACMA's CDN.

## Data Sources
| Source | URL | Licence |
|---|---|---|
| ACMA Register of Radiocommunications Licences | `https://cdn.acma.gov.au/rrl/spectra_rrl.zip` | ACMA's own terms — **not** open data, **not** CC |
| ABS ASGS 2021 SA4 boundaries | `https://geo.abs.gov.au/arcgis/rest/services/ASGS2021/SA4/MapServer/1/query` | CC BY 4.0 |
| ABS Estimated Resident Population by SA4 | `https://data.api.abs.gov.au/rest/data/ABS,ERP_ASGS2021,1.0.0/ERP.3.TOT.SA4..A` | CC BY 4.0 |

Scale: 164,105 licences · 14,373 licensee records · 2,149,290 device assignments ·
129,334 sites · 3,665 broadcast service licences · 398 spectrum licences.

## Architecture Decisions
- **Vanilla TS.** Nine views, a 40-line hash router, no shared component tree worth a framework.
- **A dependency-free ZIP reader** (`pipeline/zip.mjs`). The archive's local headers carry
  zeroed sizes with the real values in a trailing data descriptor, so a local-header walk
  reads zero bytes out of every member and reports success. It reads the central directory.
- **A dependency-free RFC 4180 parser** (`pipeline/parse.mjs`), imported by the tests, so
  the parser under test is the parser that runs. `readline` is not RFC4180-safe and a naive
  split misaligns 12,273 site rows while leaving the coordinates intact — the map looks
  perfect and every attribute is wrong.
- **One entity id space.** The register's unit is a client number; organisations hold several
  (Telstra 9, the NSW Telco Authority 10, Airservices 6). Every payload stores an entity id
  and nothing else, and a gate recomputes two bands' holder counts from the device stream to
  prove no payload has slipped back to raw client numbers.
- **Point-in-polygon runs on the raw 101 MB boundary file**, not the 1.1 MB drawing file. A
  gate asserts the two give *different* answers for one region, which is the only way to
  prove the right file was used.
- **A hand-rolled canvas layer** for 71,361 map points. 71,000 Leaflet markers is not a slow
  map, it is a dead tab.
- **No SVG wheel-zoom.** The two dense views navigate by band chips and a frequency box
  (the Ofcom pattern) rather than viewBox zoom, which would scale the axis text and
  labels along with the data and produce a wrong-looking chart.

## Views
Holders · Bands · Channels · Where · Sharing · Exclusive · On air · Limits · Method.
Each opens with one stated finding and its exact number before going dense. Every data mark
carries a hover tooltip with exact values; every dense view has a click that both selects and
leads somewhere.

## Test Results
- Tests written: **88**, across five files
- Tests passed: **88** · failed: 0
- Pipeline gates: **34** correctness + plausibility, all passing. Each correctness gate
  recomputes from a source CSV rather than comparing a payload with itself.

## Build Status
- npm install: pass
- npm test: pass (88/88)
- npm run build: pass
- Local preview: pass
- Production verification: pass — all nine views on the live URL, zero console errors, zero
  horizontal overflow at 375px, real-click drill-down confirmed

## Adversarial review — what it caught
Two Workflow passes (9 research/design agents, then 5 verification agents, all Opus 5) ran
against the raw CSVs with their own parsers, forbidden from importing anything from
`pipeline/`. They reproduced 42 published figures exactly and found **seven real defects**,
five of which every existing gate passed:

1. **Site suppression keyed on the client-level person flag while the roster keyed on the
   entity-level one.** Three sites belonging to an entity the site itself treats as a natural
   person were published — one a suburban residential address.
2. **The name-shape test missed joined partnerships.** "GW & DJ Lewis" is two people, and
   the `&` normalises to " AND ", pushing 110 couples past the word limit. 69 held published
   sites; 19 were sole-held and named after a street address.
3. **`halves.every(bareNameShape)` passed Array.every's INDEX as the minimum word count**, so
   the third part of a three-way name needed two words. Seven more families leaked.
4. **Band top-holder tables credited each holder's whole-register device count to every band
   its licence touched** — 30% of cells wrong, three rendering a "share of band" above 100%.
5. **`health.json` read a Map with property syntax**, publishing a 0% bar where the true
   figure is 66% — the correct number two keys away in the same payload.
6. **The Bands headline compared two bands on device ROWS**, which inverts under the site's
   own de-duplication rule: on rows 3.5 GHz looks 2.7× larger than land mobile; on distinct
   assignments land mobile is twice the size.
7. **The megahertz correction was welded to hard-coded numbers** that no longer matched the
   dynamic ones beside them ("32 market areas" against a 780× figure).

All seven are fixed and six new gates now cover them (G8a/b, G10b, G17, G18, G18b, G5e).
Also fixed from the same review: the conditions table is now downloaded and measured rather
than asserted; the receiver count is counted rather than inferred; the per-capita ramp, the
legend breakpoints, both year axes and the megahertz correction are derived from the data;
the boundary file carries its ABS credit; and the Method page no longer claims there is no
search box when there is one over site names.

## Errors & Resolutions
- **Data pipeline dispatch, HTTP 403.** The first CI run committed data successfully and then
  failed dispatching `deploy.yml`, because the job declared only `contents: write`. Added
  `actions: write`. This is the fleet-wide "a pipeline commit never deploys" defect: a push
  made with `GITHUB_TOKEN` does not trigger another workflow, so without the dispatch the
  data lands, the job goes green and the live site never changes.
- **CI reproducibility.** The first pipeline run reproduced every payload byte-identically
  except `sa4.geojson`, whose boundary simplification differs in the last decimal place
  between mapshaper on macOS and on the runner. Everything derived from the register itself
  matched exactly.
- **The Browser pane's screenshot compositor** renders a torn frame at any scroll offset in
  this environment, and the user's Chrome cannot reach the sandboxed preview server. Views
  below the fold were verified by hiding sibling panels with page JavaScript and shooting at
  scroll zero; every assertion (overflow, overlay dismissal, click effects) was made against
  computed state rather than the image.
- **The first arc diagram was a fabrication.** It drew each duplex spacing from an invented
  "representative" frequency, which rendered as a row of spikes and would have been a made-up
  picture of a real structure. The pipeline now emits the 900 commonest real (transmit,
  receive) pairs and the arcs are drawn from those.

## Deployment
- Repo created: yes — https://github.com/ben-gy/au-spectrum
- GitHub Pages enabled: yes, custom domain + TLS certificate approved
- Directory entry live on main: yes — confirmed being served from
  `ben-gy.github.io/gh-site-factory/index/sites.json`
- Workflow triggered: yes — deploy succeeded; the monthly data pipeline ran end-to-end in CI
- IndexNow pinged; the lab hub redeployed

## Known limitations, stated honestly
- **The suppression rule is a heuristic on top of the register's own flag.** 37 named entities
  still carry person-shaped names and survive only because they hold an ACN or more than five
  licences. The Method page says so.
- **The merge occasionally joins two organisations that share a name** — Central Coast Council
  in Tasmania and in NSW are one holder here. Measured at roughly 200 licences, 0.13% of the
  register, none in the top fifty. Disclosed on the Method page.
- **Several gates pin exact counts** (2,149,290 device rows, 73,501 live sites, 3,665 BSLs).
  Every one of these changes on the ACMA's next daily rebuild, so the monthly run will fail
  loudly and a human will update the constants. That is deliberate — a tripwire, not an
  oversight — but it does mean the update path is not unattended.
- **1,246 sites (0.96%) fall outside every SA4 polygon** and are absent from the regional
  aggregates. 411 of them cannot be in one by construction (external territories, Antarctic
  stations); the rest are coastal or offshore points.
