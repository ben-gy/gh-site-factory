# Build Log: Warming
**Date:** 2026-08-01
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty. Surveyed the 69-site registry (very AU-heavy) for an
uncovered domain and found **no climate / long-term temperature / weather** site
(au-dams is water storage, aurora-australis is space weather, quakewatch is geohazards).
Verified data accessibility before committing: BOM ACORN-SAT (the official homogenised
temperature record) is downloadable per-station with a browser UA. The easy
`/climate/change/acorn/sat/data/acorn.sat.*.daily.txt` URL is the STALE v2.1 (2018,
ends 2017); the CURRENT data lives at `/climate/change/hqsites/data/temp/tmax|tmin.{id}.daily.csv`
(found via the station page's `acorn-sat.js`), 112 stations all reaching 2024-12-31,
plus `acorn_sat_sites.csv` for metadata. Genuinely new, authoritative, annually-updated,
richly-visualisable → built it.

## Site Details
- **Name:** Warming
- **Repo:** ben-gy/au-warming
- **Category:** environment (index category: data-explorers, country AU)
- **Audience:** Ordinary Australians asking "has my town got hotter, and by how much?"; plus teachers, students, journalists.
- **Stack:** Vanilla TypeScript + Vite + Vitest; Leaflet for the map; all other charts hand-rolled SVG.
- **Data strategy:** pipeline, **annual cron** (`23 5 11 3 *`) — BOM refreshes ACORN-SAT once a year ~Feb–Mar, the proportional/slowest-allowed cadence. Idempotent (embeds the DATA year 2024, not a timestamp).

## Data Sources
- BOM ACORN-SAT homogenised daily max & min temperature per station, 1910→2024 — `www.bom.gov.au/climate/change/hqsites/data/temp/tmax|tmin.{id}.daily.csv` (CC BY 4.0)
- BOM ACORN-SAT station metadata (112 stations: number, name, lat, lon, elevation, start) — `www.bom.gov.au/climate/data/acorn-sat/stations/data/acorn_sat_sites.csv` (CC BY 4.0)
- ABS ASGS state & territory boundaries — vendored `patterns/geo/au-states.geojson` (CC BY 4.0)

## Architecture Decisions
- **Vanilla TS** — single-page hash-routed tabs, no component tree; smaller/simpler than React.
- **Pipeline, not realtime** — an annually-published dataset; the browser fetches pre-computed compact JSON (`climate.json` 1.3 MB for annual series + 112 lazy `stations/{id}.json` for the year×month heatmaps). Dependency-free `pipeline/parse.mjs` is the tested core; `collect.mjs` fetches with a browser UA + retry + a 6-way pool.
- **State tagging by point-in-polygon** against the vendored ABS polygons (offshore islands fall back to nearest polygon) — no hand-authored geography.
- **Single diverging anomaly ramp** (blue→red) reused across every view so colour always means cooler/warmer-than-1961–1990.
- **Three reconciliation gates** for a derived dataset: two-way annual-mean cross-check; raw-value sanity range; and a national-trend ANCHOR to BOM's published ~1.5 °C / ~0.13 °C-per-decade figure (the strongest validation available for computed data).

## Test Results
- Tests written: 52 · passed: 52 · failed: 0
- Coverage: daily-CSV parser (sentinel/blank drops, malformed rows), aggregateStation (means, 1961–1990 anomalies, hot-day/frost counts, two-way gate), olsTrend/trendPerDecade slope recovery, assignState point-in-polygon (inside + offshore fallback), buildNational, all three gates INCLUDING injected-error cases, positional layout (squarify at 112-station scale + histogram binning), and a jsdom render of all 7 non-map views against the real committed data (no NaN/undefined/Infinity + signature marks).
- Two issues found & fixed via tests: `new URL(...,import.meta.url)` isn't a file:// URL under vitest (→ read from cwd); `attachSvgZoom` needs a jsdom guard (`viewBox.baseVal` absent).

## Build Status
- npm install: pass
- npm test: pass (52/52)
- npm run build: pass (tsc + vite; 210 KB JS / 65 KB gz; only Leaflet ships)
- Local preview: pass (all 9 views, zero console errors)

## Verification (in-browser, real trusted clicks against the byte-identical production build)
- All 9 views render with zero console errors.
- Map: 112 station markers over faint real state polygons (in a lower pane so they never cover markers); a REAL marker click opens the station drawer painting ABOVE Leaflet; the About modal opens above Leaflet FROM the map view (z: map-frame isolate 0, boundaries pane 250, drawer 2000, modal 2100, gloss-pop 2300, hover-tip 3000).
- Explorer: a REAL histogram-bar click filtered the table 112→45 with an active bar + filter pill.
- Drawer works on desktop and at 375px (full-width); deep-linkable (#s=id).
- Zero horizontal overflow at 375px across ALL 9 views AND with an open drawer (asserted: scrollWidth==clientWidth==375).
- Fixes made during verification: rebuilt the drawer/modal entrance from a rAF-triggered CSS transition (throttled → stuck off-screen in a backgrounded tab) to a CSS @keyframes whose resting state is on-screen; clear stale tooltip/modal on view navigation; map legend rebuilt as swatch+label chips; jsdom guard in attachSvgZoom.

## Deployment
- Repo created: yes (ben-gy/au-warming)
- GitHub Pages enabled: yes (workflow build); Cloudflare CNAME `au-warming` → ben-gy.github.io created; Pages CNAME set + cert cycle triggered.
- Deploy to GitHub Pages workflow: GREEN on first run (npm ci + npm test + build + deploy).
- Data Pipeline workflow: GREEN on the initial push (paths: pipeline/**) — collect.mjs fetched the live BOM feed on CI, all 3 gates passed, and it was idempotent (main head unchanged, no data commit).
- Production: HTTPS returns 200; live bundle `index-D9WE7zpY.js` MATCHES local dist byte-for-byte; all data/SEO endpoints 200.
- PR created: https://github.com/ben-gy/au-warming/pull/1 (assigned ben-gy)

## Key data facts (from the real BOM pipeline)
- National warming since 1910: **+1.46 °C** (rate 0.13 °C/decade) — matches BOM State of the Climate.
- 2024 the hottest year (+1.4 °C vs 1961–1990). Nights (+0.14) warming faster than days (+0.11).
- 35 °C+ days: ~36 → ~49 a year (average station). Frost nights: ~21 → ~17.
- Fastest-warming station: West Wyalong NSW (+0.29 °C/decade). Hottest day in the record: Oodnadatta SA 51.1 °C (homogenised).

## Errors & Resolutions
- **Stale-bundle red herring:** early map debugging showed "0 markers" — the in-app browser had cached an old `index.html`; cache-busting (`?v=`) loaded the fresh bundle and all 112 markers appeared. (Also restructured the map so markers draw synchronously, independent of the boundaries fetch, and boundaries sit in a lower pane — an improvement either way.)
- **Drawer/modal stuck off-screen** in the backgrounded verification tab — CSS transitions/animations freeze there; switched to a @keyframes entrance with an on-screen resting state (see Verification). Real-user tabs animate normally; the resting state guarantees visibility regardless.
- **LIMITATION:** the live production HTTPS URL could not be loaded in the in-app Browser pane (benrichardson.dev is blocked there by policy — navigate timed out). Verification therefore ran against the byte-identical local production build (same D9WE7zpY hash now confirmed live), so live behaviour equals verified behaviour. `.https_enforced` still read false at hand-over though HTTPS serves 200.
