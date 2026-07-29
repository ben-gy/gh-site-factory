# Build Log: Aurora Australis
**Date:** 2026-07-29
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty (only commented examples). Scanned the registry (65 sites,
heavily AU government/open-data) and the live `gh repo list` — no space-weather, aurora, or
Sun-Earth site existed. Considered fuel prices (rejected: NSW FuelCheck needs a secret API
key, failing the realtime keyless constraint), earthquakes (USGS, viable but generic), and
product recalls. Chose **Aurora Australis** — a live Southern Lights forecast — because it is
(a) a genuinely new domain, (b) a strong realtime/live-dashboard candidate (the NEMWatch/
DSN-Watch class the factory prizes), (c) highly searchable and uniquely AU-relevant, and
(d) rich in visualization (gauge, oval map, forecast heatmap, solar-wind plots, city
leaderboard). Verified up-front that every NOAA SWPC feed is public, keyless and returns
`Access-Control-Allow-Origin: *`, satisfying the hard "client-side realtime" constraint.

## Site Details
- **Name:** Aurora Australis
- **Repo:** ben-gy/aurora-australis
- **Category:** science (registry) / live-dashboards (public index)
- **Audience:** Australian aurora chasers, astrophotographers and stargazers — phone-first, at night, in the field.
- **Stack:** Vanilla TypeScript + Vite + Leaflet + Vitest
- **Data strategy:** realtime (client-side fetch + poll; NO scheduled Actions — the only workflow is the push-triggered deploy). Poll cadence proportional to source: solar wind ~1 min, OVATION oval ~5 min, Kp ~10 min, alerts ~10 min, forecast text ~30 min.

## Data Sources
- NOAA SWPC Planetary K-index (observed) — services.swpc.noaa.gov/products/noaa-planetary-k-index.json
- NOAA SWPC Kp forecast — services.swpc.noaa.gov/products/noaa-planetary-k-index-forecast.json
- NOAA SWPC 3-day forecast text — services.swpc.noaa.gov/text/3-day-forecast.txt
- NOAA SWPC OVATION aurora — services.swpc.noaa.gov/json/ovation_aurora_latest.json
- NOAA SWPC real-time solar wind — services.swpc.noaa.gov/json/rtsw/rtsw_mag_1m.json + rtsw_wind_1m.json
- NOAA SWPC alerts — services.swpc.noaa.gov/products/alerts.json
- ABS ASGS 2021 state boundaries — vendored from patterns/geo/au-states.geojson

## Architecture Decisions
- **Vanilla TS** — a single-page tabbed dashboard; no component tree or routing that would justify React.
- **Realtime, no pipeline** — the displayed values are "what is happening right now", so the browser polls the public feeds directly. No GitHub Actions cron of any kind (per the realtime rule).
- **Leaflet + canvas image-overlay** — the OVATION oval is a ~65k-point lon×lat grid; rendering it as one client-generated canvas `imageOverlay` (rather than thousands of markers) keeps it smooth and re-generates each refresh. Real ABS state boundaries for context; the map is isolated (`isolation:isolate; z-index:0`) so Leaflet panes can't escape over the drawer/modal.
- **Interpretation layer** — pure `cityChance(kp, threshold)` turns raw Kp into a per-location chance using each place's community-standard minimum Kp + approximate geomagnetic latitude, stored as documented reference constants in `src/data/locations.ts` (not fabricated geometry).
- **All charts hand-rolled SVG** (gauge, bars, lines, heatmap); positional maths isolated in `src/layout.ts` for testing.

## Test Results
- Tests written: 61 (model 33, layout 22, render 6)
- Tests passed: 61
- Tests failed: 0
- Notable: the render suite caught two NaN leaks in the empty-data path — `chanceFill(NaN)` and the Kp-gauge needle geometry with a non-finite Kp — both fixed (guard to 0 / finite fallback), then green.
- Extra: a Node end-to-end harness fetched today's REAL NOAA feeds through the compiled model — current Kp 1.67 (Quiet), peak 4.33, Bz 0.59 nT, speed 397 km/s, 65,160 oval points reaching −50° lat, 125 alerts, correct city gradient — all live assertions pass.

## Build Status
- npm install: pass
- npm test: pass (61/61)
- npm run build: pass (192 KB JS / 59 KB gzip incl. Leaflet; 34 KB CSS)
- Local preview: pass (HTTP 200); byte-identical bundle deployed live

## Deployment
- Repo created: yes (ben-gy/aurora-australis)
- GitHub Pages enabled: yes (Actions build_type); Deploy workflow GREEN on first run
- DNS: yes — Cloudflare CNAME aurora-australis → ben-gy.github.io (DNS-only)
- Custom domain: HTTP 200 live; HTTPS cert still issuing at hand-over (https_enforced=false)
- PR created: https://github.com/ben-gy/aurora-australis/pull/1
- Workflow triggered: yes (success)

## Errors & Resolutions
- **3 unused-var TS errors** on first build (`arcPath` import, `windTile`/`panel` label params) — removed the unused bindings; rebuilt clean.
- **2 NaN leaks** in the no-data render path — guarded `chanceFill` and the gauge geometry.
- **Live in-browser verification blocked** — the in-app Browser pane blocks localhost and the benrichardson.dev domain by policy; the connected Claude-in-Chrome extension hung on an unapprovable side-panel permission prompt (autonomous run); the TLS cert was still issuing. Mitigated by the 61 tests + the live feed+model check + the byte-identical deploy + a static review of the overflow/z-index/sticky-footer invariants. Flagged in the PR and registry for an HTTPS re-verify once the cert flips.
