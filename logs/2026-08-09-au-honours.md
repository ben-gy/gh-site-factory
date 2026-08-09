# Build Log: Honours
**Date:** 2026-08-09
**Status:** deployed

## Idea Source
Researched. `IDEAS.md` was empty, so eight candidate domains were investigated in parallel
(schools, retail energy plans, ATSB transport-safety investigations, homelessness, ambient air
quality, business demography, maritime/ports, and a wildcard), each required to actually
*download* its data rather than describe it. Three independent judges then scored the reports.

That first round went **3–0 for business demography** with ATSB runner-up, and it disqualified the
schools idea on a licence carve-out worth recording: ACARA's school files download with no key at
all and ACARA's general copyright page does say CC BY 4.0, but the Data Access Program page
releases them "under the My School terms of use", whose clause 6.4 forbids reproduction "on a
publicly accessible website". Two agents reached opposite conclusions on this; the pessimistic one
was right, and all three judges verified it first-hand. Logged to `EXPANSION_IDEAS.md` as a
do-not-build.

**The winner came from outside that round.** `EXPANSION_IDEAS.md` already carried a curated
"not-yet-built, genuinely new" list, and the Australian honours register sat at the top of it with
a verified record count. Probing it directly confirmed a live, keyless API, so a second head-to-head
was run — business demography vs ATSB vs honours — preceded by a dedicated licence-and-robots
audit, given what had just happened to the schools idea. That audit came back
`open-reuse-permitted`: CC BY 4.0 with a closed three-item carve-out (Coat of Arms, department
logo, third-party content), **no terms-of-use page anywhere**, robots.txt unrestrictive on both
hosts, and — decisively — PM&C's own documentation advertising a CSV/Excel bulk export of search
results, making bulk extraction a product feature rather than a tolerated side-channel. The
tiebreak went **3–0 for honours**, on an empty domain, a searchable public audience, and a corpus
of named unit records with free text that supports far more distinct views than aggregate count
tables.

## Site Details
- **Name:** Honours
- **Repo:** ben-gy/au-honours
- **Category:** civic-recognition (a new domain — zero of the previous 69 sites touched honours or awards)
- **Audience:** Australians looking up a relative or a local recipient (the highest-volume real use,
  spiking twice a year when the Australia Day and King's Birthday lists are gazetted); journalists
  and researchers; community organisations who nominate people
- **Stack:** vanilla-ts + Vite 6, Leaflet 1.9.4 as the only runtime dependency, every chart hand-rolled SVG
- **Data strategy:** pipeline. **Quarterly cron (Feb/May/Aug/Nov)** — the register publishes two big
  lists a year plus continuous bravery/meritorious gazettals, so quarterly catches both lists within
  weeks and picks up rolling awards in between. Never faster than monthly.

## Data Sources
- PM&C "It's an Honour" register — `POST https://honours.services.pmc.gov.au/api/search/v1`
  (keyless Azure Cognitive Search passthrough, discovered by reading the SPA bundle). 289,683
  records, 1877–2026. CC BY 4.0, © Commonwealth of Australia 2023.
- ABS Census 2021 DataPack **G01** by Postal Area — population and age structure.
- ABS Census 2021 DataPack **G02** by Postal Area — median household income and median age.
- ABS ASGS 2021 Postal Areas — 2,641 polygons, 107,541 vertices after mapshaper `-simplify 1.5%`.

## Architecture Decisions
- **Vanilla TS** — eight views, one hash router, no component tree worth a framework.
- **Server-side pipeline, not runtime fetch** — the honours API is CORS-pinned to
  `honours.pmc.gov.au`, so a browser can never call it. Static JSON is the only option, and it is
  also the right one.
- **Search sharding.** 289,683 records will not ship as one file. Records are indexed under the
  initial of **both** surname and given name (so either token finds a person) across 26 shards,
  with repeated strings interned into dictionaries and citations moved into 32 **id-keyed** buckets.
  The first cut, with citations inlined and duplicated per shard, was 86 MB; this is 49 MB with the
  largest shard at 748 KB gzipped and the largest citation bucket at 92 KB.
- **No per-person route.** A deliberate constraint, not an oversight — see Privacy.

## Findings, and what happened to them

**Shipped.** Sorting every postcode by median household income, the bottom **eight** deciles receive
Order of Australia awards at roughly the same rate (4.70–6.75 per 10,000, a spread of 1.44×); only
the top two break away, at 10.96 and 16.88. The poorest decile (6.67) out-rates the third, fourth,
fifth, sixth, seventh and eighth. The shape is a **J, not a slope**, and describing it as "richer
postcodes get more honours" or "poor postcodes are overlooked" would both be wrong.

**The obvious objection was tested and failed in the opposite direction.** Recipients skew elderly
and wealthy suburbs might simply be older. Re-expressed per 10,000 residents **aged 65+**, the
top-to-bottom ratio *widens* from 2.53× to 4.45×, because the richest postcodes are **younger**
(mean median age 40.2) than the poorest (49.2). Age was suppressing the gradient, not producing it.

**Killed before any view code was written.** "The gender gap widens as the honour gets higher" is
widely believed and does not survive this data. The register publishes no sex field; the only
instrument is the gazetted honorific, and it fails hardest exactly where the claim would be
strongest — unable to classify **65.2% of Companions** against **16.3% of Medal recipients**,
because senior figures are gazetted as Professor, Dr or by rank, and that residual is itself
sex-skewed. The all-time figures *look* monotonic (23.2 / 25.4 / 30.9 / 39.1%), which is the trap;
recent years are not. The "Who" view therefore draws the arithmetic **bound** rather than an
estimate — Companions span 8.2%–73.4%, the Medal 32.7%–49.0% — and the overlap visibly refuses to
rank the levels.

**Corrected by the data mid-build.** The design brief asserted the Order of Australia displaced
Imperial honours from 1975 and that Imperial ended in 1992. Both wrong: the two ran in **parallel
for eight years**, the Order only overtook Imperial in **1983** (443 v 58) after a near dead heat in
1982 (530 v 561), and one Imperial record sits in **2000** (Royal Victorian Order — in the
Sovereign's personal gift, never Australia's to withdraw). The glossary had been written with the
wrong 1992 claim; the view agent flagged that its own chart would contradict it on screen, and it
was corrected before launch.

## Traps found in the real data (all gated)
1. **Award systems are double-coded** — `Order of Australia`/`OA`, `Meritorious`/`ME`, `Realm`/`RE`,
   `Bravery`/`BR`, `Imperial`/`IM`. Not merging undercounts the Order of Australia by **3,832
   (7.6%)**. `IM` has exactly **one** record, so a merge map written by eyeballing a facet list
   drops it silently — the gate therefore asserts that **no abbreviation survives**, rather than
   checking the four pairs someone happened to notice.
2. **`GazetteState` case variants** — `VIC` (13,637) and `Vic` (4,764) are the same state. A
   case-sensitive grouping loses **26% of Victoria**, 27% of Queensland and 22% of Tasmania: enough
   to reorder a state ranking, small enough to look plausible. 231 raw strings → 9.
3. **1,430 "citations" are gazette filing references** (`AM (CIVIL DIVISION) QB 1975`), excluded
   from the taxonomy rather than shown to a reader as a description of what someone did.
4. **Three ABS pseudo-postcodes** (`9494`, `9797`, `ZZZZ`) ship with **null geometry** and a
   non-numeric code — they crash the renderer and break the postcode join.
5. **`$skip` caps at 100,000**, so a naive page-through of 289,683 dies two-thirds in. Sliced by
   award year, every slice asserted under the cap first.
6. **`HonoursListName` is unnormalised free text** — 196 raw strings for six occasions; "Australia
   Day", "Australia Day 2019" and "Australia Day 2020 Honours List" are one list.
7. **A latent bug in my own code**, caught by a test rather than by the data: a name containing no
   letters returned an empty shard list, which would have indexed the record in **no shard at all**
   — present in every count, permanently unsearchable.

## Test Results
- Tests written: **72** (37 parser, 22 against the real committed data, 13 positional layout)
- Tests passed: **72**
- Tests failed: 0
- Three failed during development and all three were real: the null-geometry pseudo-postcodes, the
  no-letters shard bug above, and a test whose own expectation was wrong about how digit-leading
  tokens are indexed.
- **20 pipeline gates**, all passing. Three assert a *limitation* remains true rather than a number
  being right: `geography-is-oa-only`, `national-medal-dominates` (41.7% of the register is one
  15-year long-service medal), and `gender-instrument-fails-at-the-top`. Two assert the headline
  survives its own objection: `income-gradient-survives-age-control` and
  `gradient-is-J-shaped-not-linear`.

## Build Status
- npm install: pass
- npm test: pass (72/72)
- npm run build: pass
- Local preview: pass

## Deployment
- Repo created: yes — https://github.com/ben-gy/au-honours
- GitHub Pages enabled: yes
- DNS: yes — `au-honours.benrichardson.dev` CNAME → `ben-gy.github.io`. **This consumed the
  benrichardson.dev zone's last free-plan DNS record (now 200/200).** The next site cannot get a
  subdomain without reclaiming one; 19 CNAMEs in that zone point at `ben-gy.github.io` with no
  matching repo and are reclaimable, but deleting the user's DNS records was not mine to do.
- TLS certificate: **approved** on the second poll.
- Deploy workflow: **success** (run 31317675878).
- Directory entry live on main: yes.
- Pull request: none, per the routine.

## Verification
**Live production URL, by HTTP:** `/`, `data/meta.json`, `core.json`, `geo.json`, `poa.geojson`,
`names/S.json`, `cites/0.json`, `og.png`, `robots.txt`, `sitemap.xml`, `favicon.svg`,
`third-party-notices.txt` and the IndexNow key all return **200**. The live bundle
(`index-CZR1Gxfs.js`) **matches** the local `dist`. Live `meta.json` reports 289,683 records, as-at
2026-07-30, 20 gates, all passing.

**Browser verification ran against the byte-identical local build of the same `dist`**, because the
in-app browser pane became unresponsive after the deploy and the Chrome extension needs interactive
approval that an unattended run cannot give. This is stated plainly rather than glossed:
*production itself was verified by HTTP and bundle hash, not by a browser.* What the browser did
confirm, on the identical bundle:
- All **eight** views render with **zero** console errors, **zero** NaN/Infinity/undefined in any
  SVG coordinate attribute, **zero** literal `&#10;` entity leakage, and **zero** native `<title>`
  elements used as tooltips.
- **No page-level horizontal overflow** at 1280px *or* at a true 375px viewport on any view —
  asserted by attempting `window.scrollTo(200, …)` and reading `scrollX` back as **0**, not by
  eyeballing a screenshot. Wide tables, matrices, the icicle and the Leaflet canvas all scroll
  locally inside their own containers.
- **The full dismissal contract passes**, asserted by computed style and re-opened between each
  exit: the ✕ (a real `<button>`, so Enter and Space reach it), the scrim by mouse `pointerdown`,
  the scrim by a real `pointerType:'touch'` sequence with no double-fire reopen, and Escape — plus a
  clean baseline (no `[hidden]` element rendering, `elementFromPoint` at the viewport centre
  returning page content) and the scroll lock released afterwards. Verified on both the About modal
  and the recipient drawer.
- **The About modal opened from the map view paints above Leaflet** (`elementFromPoint` lands inside
  the modal; `.map-frame` carries `isolation:isolate; position:relative; z-index:0`).
- **Real trusted clicks**: an example-search pill, and a search result row that opened the drawer for
  a named recipient with the correct citation.
- Searching "nguyen" returns **48** records; the drawer shows the real citation; and the URL stays
  `#/search?q=nguyen` — **no per-person URL is ever created**.
- The map renders **2,641 real ABS polygons** with CARTO tiles and full OSM/CARTO/ABS attribution.

**One real visual defect was found by looking at a screenshot and fixed.** The "Who" view authored
its charts in a 370-unit coordinate space with `width="100%"`, so on a desktop they upscaled **3.3×**
and rendered 11px labels at **36px**, overlapping the bars beneath them. Every other view was then
swept programmatically for the same fault (effective font size = declared size × viewBox scale);
only that view was affected. Fixed by authoring both charts near their rendered width, rebuilt, and
re-verified visually.

## Errors & Resolutions
- **ABS SDMX returned an unrecognised shape** — the G01 request needs
  `Accept: application/vnd.sdmx.data+json`; without it the response has no `data.structures` and the
  parse fails on an undefined read rather than on anything that looks like a content problem.
- **mapshaper field name** — the ArcGIS export lowercases to `poa_code_2021`, not `POA_CODE_2021`.
- **`parse.mjs` types** — TypeScript resolves a `.mjs` import to `parse.d.mts`, not `parse.d.ts`.
- **Browser tooling instability** — the in-app pane rendered only on a freshly-created tab and went
  blank (`visibilityState: 'hidden'`) after the first screenshot, so each view was captured on its
  own new tab; the Chrome extension timed out entirely. Measurements taken while the pane was
  backgrounded report `innerWidth: 0` and must be discarded — one intermediate reading was, before
  it could produce a wrong conclusion.

## Honest limitations, all stated on the site
- **72% of the register has no geography.** A postcode is published for 24.7% of all records but
  87.5% of Order of Australia records, so every map and per-capita figure here is an **Order of
  Australia** measure and says so on the artefact itself.
- **41.7% of the register is one award** — the National Medal, for 15 years of service. "Most common
  honour" is never presented as "most honoured achievement".
- **Service domains are an editorial classification** derived here from citation text, not published
  by the register; 5.6% fall to "other" and it is treated as a residual, not a field.
- **Awards sit where the recipient was gazetted**, which need not be where they did the work, and
  are compared against a 2021 Census denominator (hence the 2010+ restriction).
- **CBD postcodes distort the age-controlled rate** — very few residents over 65, so the denominator
  is unstable. Called out above the map rather than left to be discovered.
- **The sex signal is a weak instrument** and no claim is made that it cannot support.

## Licensing
AGPL-3.0-or-later with the section 7(b) attribution term; `LICENSE`, `ADDITIONAL-TERMS.md`,
`CONTRIBUTING.md`, `THIRD-PARTY-NOTICES.md` + `public/third-party-notices.txt`, SPDX headers on
every first-party source file, `"license": "AGPL-3.0-or-later"` in `package.json`. The only
third-party runtime component is **Leaflet 1.9.4 (BSD-2-Clause)**, confirmed as the sole
`node_modules` entry in the production sourcemap. No third-party fonts. PM&C CC BY 4.0 and ABS
CC BY 4.0 attribution appears in the footer, the About panel and the JSON-LD; the Commonwealth Coat
of Arms and the department logo are excluded from the licence and are not reproduced anywhere.
