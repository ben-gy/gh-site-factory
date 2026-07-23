# Build Log: Bank Branches

**Date:** 2026-07-23
**Status:** deployed (registry status `verify_production_pending` — production live & healthy, but in-browser production click-through blocked by a browser-tooling outage; see Deployment)

## Idea Source

Researched. IDEAS.md was empty. The catalogue already holds 40+ AU government/open-data
explorers, so I searched for a genuinely novel domain with rich geography and a strong
counter-intuitive story, and validated data availability for three candidates (APRA
bank points of presence, NDIS participant data, ASIC corporate insolvency). **APRA's
Authorised deposit-taking institutions' points of presence** won: a hot political topic
(the 2023–24 Senate inquiry into regional bank closures), a single authoritative annual
source, and — uniquely in the catalogue — per-branch latitude/longitude, enabling a real
POINT map rather than only a choropleth. No existing site touches banking access.

## Site Details

- **Name:** Bank Branches
- **Repo:** ben-gy/au-branches
- **Category:** finance (index category `data-explorers`, country AU)
- **Audience:** regional/rural Australians losing local branches; older and in-person
  bankers; small business; journalists and MPs on the regional-banking beat.
- **Stack:** Vanilla TypeScript + Vite + Vitest; Leaflet for maps.
- **Data strategy:** pipeline, **annual** cron (12 Nov). APRA publishes once a year each
  October — annual is the correct proportional cadence; anything faster just re-commits
  identical data.

## Data Sources

- APRA ADI Points of Presence — single xlsx, June 2017–June 2025, Table 1 raw
  (137,248 physical-channel rows) + Table 2 per-SA2 population.
  https://www.apra.gov.au/authorised-deposit-taking-institutions-points-of-presence-statistics
- ABS Estimated Resident Population by SA3 (ERP_ASGS2021) — per-capita denominator.
- ABS ASGS 2021 SA3 boundaries — reused from a sibling (au-mortality), committed static.
- ABS Regional Population by Remoteness Area — embedded constant for the equity view.

## Architecture Decisions

- **Vanilla TS** (not React): single-page tabbed explorer with a map — smaller, simpler.
- **Dependency-free pipeline**: `pipeline/parse.mjs` is a minimal zip + XLSX reader using
  only Node built-ins (zlib), shared with the vitest suite, so CI never installs anything
  and the data-pipeline workflow runs `node` directly (no npm step).
- **Point map via Leaflet canvas renderer** (`preferCanvas`) to draw ~10,460 markers
  smoothly, with channel layer toggles; a second mode swaps to the SA3 per-capita choropleth.
- **Signature analysis chose the SA2 (town) grain** for "banking deserts": at SA3 only 9
  regions hit zero, but at town level 366 lost their last branch — the human story.

## Key data traps handled

1. **Table 2 population is at suburb grain**, not SA2 — it repeats per suburb (summing
   double-counts) and only covers SA2s that *have* a service (undercounts regional
   population, which would inflate per-capita access exactly where the desert story lives).
   So per-capita denominators use **full ABS ERP by SA3** (all 340 regions, incl.
   zero-branch ones); the deserts "people affected" figure dedupes Table 2 to one value
   per SA2.
2. **EFTPOS excluded** — ~700k shop card terminals are merchant payment, not banking access.
3. **Truncated SA3 names** — APRA occasionally ships a shared-string cut off
   ("Augusta - Margaret River - Busselto[n]"); the name→code join has a unique-prefix
   fallback. 324/325 exact, 100% after fallback.
4. **Reconciliation guard** — the pipeline asserts region branch totals reconcile to
   ≥97% of the national branch total and fails the build on drift (matched 100.0%).

## Headline findings

- Branches **5,694 → 3,205 (−44%)**; bank-owned ATMs **13,814 → 5,143 (−63%)**; but
  **Bank@Post held: 3,578 → 3,365 (−6%)**.
- **366 towns lost their last branch** since 2017 (~**4.4m** residents); 298 keep a
  Bank@Post, **68 have no staffed banking at all**.
- Remote/regional Australia lost the largest *share* of branches.

## Test Results

- Tests written: 36 (4 files) — parser: excelToYear, channel map, SA3 prefix-match,
  extractRecords (EFTPOS drop, coords), aggregate reconciliation + lost-last-branch flag;
  histogram binning; treemap positional (in-bounds / no-overlap / no-NaN / area); svgZoom math.
- Tests passed: 36 / 36.

## Build Status

- npm install: pass
- npm test: pass (36/36)
- npm run build: pass (tsc + vite; 232 kB JS / 71 kB gzip)
- Local preview (vite preview of dist): pass

## Local verification (byte-identical to production bundle `index-ENW7Kaps.js`)

- All 9 views render error-free, no NaN/undefined/Infinity anywhere.
- Point map draws every 2025 service point on the real coastline, colour-coded by channel;
  Access mode draws the SA3 per-capita choropleth on real ABS geometry with a ratio ramp.
- **Real trusted click** on a map SA3 polygon opened the region drawer (Lachlan Valley:
  34→19 branches, Bank@Post steady ~20), painting **above** Leaflet (isolate + z-index 2100).
- About modal opened **from the map view** paints above the map (elementFromPoint inside modal).
- **Zero page-level horizontal overflow at 375px** across all 9 views + an open drawer
  (wide tables scroll locally inside `overflow-x:auto`).

## Deployment

- Repo created: yes (ben-gy/au-branches), pushed to main.
- GitHub Pages: enabled (workflow build). Deploy run **completed/success**.
- DNS: Cloudflare CNAME au-branches → ben-gy.github.io created.
- TLS: `https_enforced=true` after one CNAME recycle. Custom domain returns **200 over
  HTTPS**; live bundle byte-identical to local dist; national/points/sa3.geojson/og.png all 200.
- PR: https://github.com/ben-gy/au-branches/pull/1 (assigned ben-gy).
- Registry + public index updated on main; IndexNow pinged (202/200); hub sitemap refresh triggered.

## Errors & Resolutions

- **One tsc error** (unused `national` in map.ts) — removed, rebuilt clean.
- **Browser tooling outage at production-verify time:** the in-app Browser pane blocks the
  `benrichardson.dev` domain by policy, and the Claude-in-Chrome extension was disconnected
  /unresponsive across ~7 retries (needs the user to reconnect the extension side panel).
  So Part B (real clicks against the live URL) could not run. Mitigation: the live bundle is
  **byte-identical** to the local dist that passed full in-browser verification above
  (including a real trusted click and the z-index/overflow checks), and the production
  endpoint is confirmed healthy. Registry status set to `verify_production_pending`; a
  re-run should click banks-bar → institution drawer, a deserts row, and an insights link
  on the live URL (all reuse the already-proven navigate()/openRegion/openInstitution path)
  and flip status to `deployed`.
