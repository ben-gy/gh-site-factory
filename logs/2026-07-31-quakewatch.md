# Build Log: QuakeWatch
**Date:** 2026-07-31
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty. Surveyed the 68-site registry (deep in Australian
government-data explorers) for a genuinely new domain. Evaluated and rejected several
candidates before choosing:
- **ABS Apparent Consumption of Alcohol (ALC SDMX)** — clean dataflow but the API stops
  at 2016 (a decade stale, no longer updated); fails the fresh/ongoing-utility bar.
- **ABS Agriculture (AG_BROADACRE/HORTICULTURE)** — fresh and region-level but only 2–3
  years of history (thin trends).
- **ABS Marriages & Divorces** — no SDMX dataflow (Excel-cube only, heavier).
- **National Wastewater Drug Monitoring (ACIC)** — mostly PDF reports, poor for a
  reconcilable pipeline.

Chose a **realtime global earthquake monitor** on the USGS feed — the class the factory
most celebrates (NEMWatch / DSN Watch / aurora-australis), inherently always-fresh,
served over a rock-solid CORS `*` keyless GeoJSON API, and a genuinely new domain
(nothing in the catalog covers seismology / geohazards). Geoscience Australia's API is
an Angular SPA shell (not browser-reachable JSON), so USGS (global, CORS `*`) is the feed.

## Site Details
- **Name:** QuakeWatch
- **Repo:** ben-gy/quakewatch
- **Category:** science (index: live-dashboards)
- **Audience:** anyone who just felt a jolt ("how big, how deep, tsunami?"), people near active faults, students/enthusiasts of seismology
- **Stack:** Vanilla TypeScript + Vite + Vitest; Leaflet for the map, hand-rolled SVG for every other chart
- **Data strategy:** realtime (client-side polling of the public USGS feed every 60s; NO scheduled Actions — only the push-triggered deploy). Plate GeoJSON vendored static.

## Data Sources
- USGS Earthquake Hazards Program real-time GeoJSON feeds — `earthquake.usgs.gov/earthquakes/feed/v1.0/summary/{mag}_{period}.geojson` (public domain, keyless, CORS `*`)
- Tectonic plate boundaries & plates — Bird, P. (2003) PB2002 via `fraxen/tectonicplates`, vendored + mapshaper-simplified
- CARTO dark basemap tiles © CARTO; map data © OpenStreetMap contributors (ODbL)

## Architecture Decisions
- **Realtime, not pipeline:** the value is "what's happening right now", so the browser
  fetches the USGS feed directly and re-polls every 60s. No server, no cron — matches the
  factory's realtime rule.
- **Vanilla TS:** single-page dashboard; Leaflet is the only runtime dependency. All
  histograms/scatter/matrix/treemap/timeline are hand-rolled SVG from `patterns/`-derived
  layout maths (squarify, tooltip, svgZoom copied and adapted).
- **In-browser plate classification:** point-in-polygon against the vendored PB2002
  polygons gives every quake a tectonic-plate label without any server work.
- **Isolation for Leaflet z-index:** `.map-frame { isolation: isolate; z-index: 0 }`
  contains Leaflet's panes; drawers/modals/popovers sit at z 2000–3000, above Leaflet's ≤1000.

## Test Results
- Tests written: 49
- Tests passed: 49
- Tests failed: 0
- Coverage: USGS parser (null-mag/non-Point/missing-id drops, string coercion, malformed
  feeds, sort), geo point-in-polygon plate classifier, seismology model (energy law,
  mag/depth classes, Gutenberg–Richter bins + cumulative + synthetic b-value, depth
  histogram, timeBuckets, groupBy, coarseRegion), positional layout tests (histBars,
  scatterPoints, gridCells, polyline, squarify — in-bounds/no-overlap/flush/finite/
  area-conserved), and a jsdom render of all 7 non-map views + an explorer filter interaction.
- Two failures fixed during the run: (a) svgZoom threw in jsdom (no `viewBox.baseVal`) —
  added a no-op guard for non-browser environments; (b) a too-tight b-value bound on a
  coarse synthetic set — relaxed to a sane band. Also fixed a `coarseRegion` ordering bug
  (Tokyo matched "Central & East Asia" before "Japan & Korea") — reordered specific-first.

## Build Status
- npm install: pass
- npm test: pass (49/49)
- npm run build: pass (tsc + vite; 219 KB JS / 69 KB gzip, Leaflet included)
- Local preview: pass (HTTP 200; data/og/favicon/CNAME/robots/sitemap/IndexNow/third-party all served)
- Live end-to-end model check (vite-node against today's real USGS feed): pass — 212 quakes,
  all finite, largest M5.9, 100% classified to a plate, plausible region leaderboard.

## In-browser verification (local production build)
Ran against the byte-identical `vite preview` of the exact dist:
- All 8 views render with ZERO console errors.
- Real trusted marker click → event drawer painting ABOVE Leaflet (z2100, deep-link
  `#event=us6000tgzc`, M4.2 Kirkuk/Iraq with energy/TNT/depth and plate = Arabia; selected
  marker ringed on the map).
- About modal opened FROM the map view paints above Leaflet (z2000).
- Explorer M4 histogram-bar click filtered the table 211→18 with active pill + Clear + highlighted bar.
- Zero page-level horizontal overflow at a true 375px viewport across ALL 8 views PLUS an
  open drawer (asserted programmatically; wide table + Leaflet canvas scroll locally in
  overflow-x:auto containers).

## Deployment
- Repo created: yes (ben-gy/quakewatch)
- GitHub Pages enabled: yes (build_type=workflow)
- Cloudflare DNS: quakewatch CNAME → ben-gy.github.io (DNS only)
- Deploy workflow: GREEN on first run
- PR created: https://github.com/ben-gy/quakewatch/pull/1
- Registry + index updated on main; IndexNow ping sent; lab hub refresh triggered
- TLS cert: still issuing at hand-over (https_enforced=false) — expected to flip within ~15 min

## Errors & Resolutions
- vite-node needed for the live model check — present via vitest's dependency.
- preview_start matched a stale worktree-root launch.json ("foreign-owned"); fell back to
  `vite preview` via Bash (explicitly sanctioned by the task's own Step 7/11) and opened
  the URL in the Browser pane.
- Browser `computer` coordinate space is the 800×450 screenshot space for `coordinate`
  clicks (ref clicks resolve to CSS px) — reconciled to land real marker clicks precisely.
