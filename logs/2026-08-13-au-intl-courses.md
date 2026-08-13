# Build Log: International Student Courses
**Date:** 2026-08-13
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty, so a five-domain discovery fan-out (transport safety, research funding, schools/VET, live environment, law/regulators) proposed ten candidates; each was then handed to an adversarial verifier told to default to *refuted*. Two died — **au-schools** on licence (ACARA's My School terms, clause 6.4, forbid republishing on a publicly accessible website; not curable by attribution) and **au-companies** on premise (the ASIC register is a current-registrations snapshot, so its apparent registration history is a survivorship curve). A judge picked the CRICOS course register from the six survivors.

Both refusals, and the deferred runner-up (**au-spectrum**, ACMA's Register of Radiocommunications Licences), are written up in `EXPANSION_IDEAS.md` with everything needed to act on them, so this round's research is not lost.

## Site Details
- **Name:** International Student Courses
- **Repo:** ben-gy/au-intl-courses
- **Category:** education
- **Audience:** a prospective international student holding an agent's quote, on a phone, in a second language, with A$15k–40k at stake; secondarily agents, reporters and policy analysts
- **Stack:** vanilla TypeScript + Vite 6, Leaflet, no charting library
- **Data strategy:** pipeline, **monthly** cron (`41 3 9 * *`) — proportional to the source, which publishes a refreshed bulk export and one new dated snapshot around the 1st of each month

## Data Sources
- CRICOS bulk export — https://data.gov.au/data/dataset/cricos (Department of Education, **CC BY 2.5 AU**), 1,512,985-byte ZIP of four CSVs
- 60 dated monthly CRICOS snapshots from the same CKAN package, Jul 2021 → Jul 2026
- ABS ASGS 2021 Postal Areas digital boundaries (CC BY 4.0), mapshaper-simplified to 4%
- matthewproctor/australianpostcodes — postcode centroids (CRICOS records no coordinates)

## Architecture Decisions
- **Vanilla TS.** Nine views, one 55-line hash router, no shared component tree worth a framework.
- **Payloads split by when they are needed, not by subject.** `meta` + `providers` at boot (≈150 KB gz); `compare` for the default view; `places`, `courses`, `churn`, `poa.geojson` lazily. Courses ship columnar — fourteen parallel arrays rather than 26,738 objects.
- **A dependency-free ZIP and XLSX reader in the pipeline** (`pipeline/parse.mjs`, ~120 lines) rather than a dependency or a shell-out to `unzip`, so the sixty-workbook backfill has no binary prerequisite and every parser is unit-testable.
- **Snapshot digests are committed** (`pipeline/snapshots/*.json`, 1.2 MB total). Sixty full workbooks would be 250 MB in the repo for a view that only ever plots counts and medians, and the backfill now happens exactly once.

## Corrections made to the research before building
Verified against the actual files rather than inherited:
1. **The "six-month gap in 2021–22" does not exist.** It was an artefact of selecting snapshot resources by `format == "XLSX"`; six months are published as `excel (.xlsx)` and one as `XLS`. Selecting by "the resource NAME contains a date" recovers all 60. The register has exactly **one** missing month, May 2022.
2. **The fee identity is exact.** Estimated Total = Tuition + Non-Tuition holds on **26,416 of 26,416** priced rows, not "26,500 of 26,547 with 47 mismatches". It is therefore gated at equality, not at a tolerance.
3. **The comparability ladder** is 355 codes with 2+ providers (not 363) once expired registrations are excluded.
4. **An `Institution Capacity` field exists** and the research missed it — the registered ceiling on concurrent overseas students. It is published, but never without its caveat attached.

## Test Results
- Tests written: **164**, across five files
- Tests passed: **164** · failed: 0

Four genuine defects were found by tests before any browser was opened: `money0('')` and `int('')` returned `0` instead of `null` (because `Number('')` is 0, so an unrecorded fee became a recorded fee of nothing); an address-matching gate caught the normaliser failing to join `120 Spencer Street` to `120 Spencer Street, Docklands`; and a geometry gate caught the boundary simplification at 1.2% leaving a median of 8 vertices per polygon, which would have rendered CBD postcodes as lozenges.

The suites are deliberately more than behaviour tests — several assert the site's *refusals* (no enrolment implications, no quantile colour scales, no zero-filled gaps, one shared misreporting threshold between pipeline and interface) so they fail the build rather than eroding.

## Pipeline gates
**21 of 21 pass**, in two classes. Correctness: the fee identity on every priced row; 100% numeric postcodes (the tripwire for a column-shifting parse bug that still produces plausible state counts); zero orphan joins; every $/week independently re-derived from the raw CSV; quantiles monotone in every group; flagged rows present but excluded from percentiles; gap months null rather than zero; the address matcher refusing four near-miss pairs it must not join; real ABS geometry by vertex count. Plausibility: register size, month-on-month drift, price distribution, comparability, the headline spread staying in per-week territory, geocoding rate, sector split.

## Build Status
- npm install: pass
- npm test: pass (164/164)
- npm run build: pass
- Local preview: pass

## Browser verification
Nine views clicked through on the production build. **Zero console errors** on any view. No horizontal overflow at 375 px on any view, with a modal open, with a drawer open, or after closing either. All four dismissal exits asserted by computed state on both the About modal and the drawers, including a `pointerType: 'touch'` sequence, with the modal opened **from over the Leaflet map** and confirmed topmost by `elementFromPoint`.

Four defects were found in the browser that no test had caught:

1. **The map never rendered in a background tab.** Leaflet's view was only ever set inside a `requestAnimationFrame`, which does not fire in a hidden tab — so the map had no view, drew no tiles and no layers, and threw nothing. Anyone opening the page in a background tab would have got an empty grey box. Fixed by setting the view synchronously at construction and keeping the layout-aware `fitBounds` as a bounded refinement on both rAF and a timeout.
2. **Views were rendered into a detached node** and grafted in afterwards, which is hostile to anything that measures its container. Now rendered into the attached container, with the render token doing the staleness work.
3. **Beeswarm dots were 2.7 px on a phone** — a 1000-unit viewBox scaled into a 335 px column. The swarm now sizes its viewBox to the viewport on narrow screens; dots are 9.3 px.
4. **Map markers hid the map.** `sqrt(providers) × 1.9` put a 26 px disc on every CBD postcode; being neighbours, they merged into one blob over both each other and the boundaries. Radii reduced, translucent, and hidden entirely past zoom 9 where the polygons speak for themselves.

Plus two copy defects: a subtitle without `display: block` rendering as "Postcode 3026DERRIMUT", and the About panel printing the count of nationally coded *qualifications* where it said *courses* (612 vs 10,301) — the pipeline now publishes both numbers separately with a comment on why they are easy to confuse.

**Not verified:** the Chrome input channel stopped delivering pointer events to the page part-way through the session — a fresh tab and a fresh window behaved the same, and controls that had opened overlays minutes earlier stopped responding to synthetic input. Before it failed, real trusted clicks were confirmed on a Leaflet marker (postcode drawer), the About button (modal over the map), a force-graph node (drawer + component highlight + deep link), and a drag on the zoomable SVG (panned **without** firing the node click — the pointer-capture contract). After it failed, the beeswarm dot was verified by a dispatched `pointerdown`/`pointerup`/`click` touch sequence at real coordinates, which opened the correct provider drawer and set the deep link. Its handler is the same pattern as the three verified by trusted click.

## Deployment
- Repo created: yes — https://github.com/ben-gy/au-intl-courses
- GitHub Pages enabled: yes (build_type workflow)
- Cloudflare DNS: yes — CNAME au-intl-courses → ben-gy.github.io
- TLS certificate: **approved**
- Directory entry live on main: yes
- Workflows triggered: yes (Deploy to GitHub Pages; Data Pipeline, which also exercises the full cold-start backfill on CI)

## Errors & Resolutions
| Error | Resolution |
|---|---|
| All 60 snapshot digests came back "0 providers, 0 courses" | The workbooks are not the CSVs with a different extension: each sheet opens with a title row and a "Report generated …" row before the header. Taking row 0 as the header yields a table whose every column is named `""` and whose every lookup is blank — which reads downstream as "the register was empty this month" rather than as a parse failure. The header is now located by looking for its key column. |
| Two gates failed on the first full run | Both were the gates doing their job: the address matcher was truncating on a fixed token window rather than on the street type, and the boundary simplification was too aggressive. Both fixed rather than relaxed. |
| Map blank on the second browser pass | See "Browser verification" above — the rAF-only view initialisation. |
