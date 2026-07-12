# Build Log: Charity Watch (AU)
**Date:** 2026-06-24
**Status:** deploy_failed (build + DNS complete; gh auth expired blocking `gh repo create` and PR)

## Idea Source
Researched. IDEAS.md was empty, so I scanned the registry for uncovered domains. We had political donations (au-donations) and disability provider compliance (au-ndis-watch) and aged care (au-agedcare) but no general charity-sector transparency tool. The ACNC publishes the full register plus annual financial returns as open CSVs on data.gov.au, so it was the right shape: high consumer relevance ("should I donate to this?"), rich enough for multiple views (financials + sectors + beneficiaries + geography), and a clear audience (anyone considering a donation).

## Site Details
- **Name:** Charity Watch (AU)
- **Repo:** ben-gy/au-charities
- **Category:** government-transparency
- **Audience:** Donors verifying a charity before giving (workplace giving, street collectors, year-end gifts, Facebook fundraisers); volunteers comparing organisations; researchers and journalists; boards and grant-makers benchmarking peers
- **Stack:** Vanilla TypeScript + Vite 6 + Vitest + Leaflet
- **Data strategy:** pipeline (monthly GitHub Actions cron downloads ACNC CSVs, joins on ABN, emits compact JSON)

## Data Sources
- ACNC Register of Australian Charities — https://data.gov.au/data/dataset/acnc-register (14 MB CSV, 65,841 rows)
- ACNC 2023 Annual Information Statement (AIS) — https://data.gov.au/data/dataset/acnc-2023-annual-information-statement-ais-data (35 MB CSV, 53,411 rows with full 2023 financials)
- ABS Estimated Resident Population (June 2024) — embedded constants for per-capita normalisation

After joining on ABN: 65,231 charities (after dropping rows with missing name/ABN), of which 50,539 have 2023 financials.

## Architecture Decisions

**Vanilla TypeScript over React.** The whole site is one-page tabs with imperative DOM rendering; React's component model would add weight without solving a real problem here. Bundle came out at 60 KB gzipped.

**Pipeline JSON over runtime CSV fetch.** The raw CSVs are 49 MB combined and would need parsing client-side. Pre-aggregating in a GitHub Actions cron job means the user pays no per-page-load CSV cost.

**Tabular form for the charity search index.** A naive object-per-record JSON came out at 27 MB. Switching to `{headers, rows}` with bit-fields for purposes/beneficiaries/operates dropped that to 10 MB (~3 MB gzipped on the wire). Financial detail moved to a separate `financials.json` keyed by ABN, loaded lazily only when the user opens the search tab or a drill-down.

**Eight overview/leaderboard/sector/flow/map/insights files all under 250 KB combined.** The Overview, Leaderboards, Money Flow, Sectors, Map, and Insights tabs all render off pre-aggregated files that finish loading in well under a second. The bulky `charities.json` and `financials.json` only load if the user clicks Search or a charity detail.

**Hand-rolled SVG for the Sankey flow.** No D3, no chart libraries. The flow renders directly from `flows.json` with Bézier-curve link paths.

**Leaflet for the state map with circle markers + CartoDB Positron tiles.** Sized by per-capita charity density. CartoDB tiles for the calm light-theme civic look that fits the audience.

**Civic warm palette (#0f2a4a navy, #0d7a7a teal, #fbfaf7 warm white, #a37b3b gold).** Light theme. Donors and parents and retirees aren't expecting a hacker terminal — they expect a calm, trustworthy public-service portal. Reference: ACNC's own visual language but slightly warmer.

## Test Results
- Tests written: 49 (across format helpers, bit-field decoding, search/filter logic, CSV parser)
- Tests passed: 49
- Tests failed: 0

## Build Status
- npm install: ✅ pass (116 packages)
- npm test: ✅ pass (49/49)
- npm run build: ✅ pass (after fixing 2 TS lint errors: unused `expenses` var, excluded `tests/` from tsc include)
- Local preview at port 5301: ✅ pass — HTTP 200 on index, JS bundle, CSS, all 7 data files, favicon, CNAME

Final bundle: 202 KB JS (60 KB gzipped) + 15 KB CSS (4 KB gzipped) + 22 MB JSON (lazy-loaded).

## Deployment
- Repo created: ❌ blocked — `gh repo create ben-gy/au-charities` returns "HTTP 401: Requires authentication". `gh auth login` needed.
- GitHub Pages enabled: ❌ blocked on above
- Cloudflare DNS record: ✅ CNAME `au-charities` → `ben-gy.github.io` (created via Cloudflare API)
- TLS cert cycle trick: ❌ blocked on above (needs the Pages site to exist)
- PR created: ❌ blocked on above
- Workflow triggered: ❌ blocked on above

## Verification

Couldn't run a visual screenshot — Chrome MCP not connected, computer-use needs interactive approval (this is a scheduled-task run with the user away). Falling back to HTTP-level verification, which passed:

- `index.html` serves 200 with correct title `<title>Charity Watch (AU)</title>` and the `id="app"` mount point.
- Built JS bundle parses cleanly under `node --check`.
- All 8 data files serve 200: aggregate (4 KB), leaderboards (200 KB), sectors (8 KB), flows (250 B), insights (3 KB), meta (2 KB), charities (10 MB), financials (11 MB).
- All 49 Vitest tests pass.

Sanity-checking the aggregate values against expected sector knowledge: total revenue $174 B, donations $12 B (7% of total — matches the well-known finding that donations are a small slice), government revenue $85 B (49% — also matches), 3.7 M volunteers, NT highest charities-per-capita at 340/100k (matches the long-standing per-capita pattern for Territories), Victorian Catholic Education Authority top by revenue at $3.4 B (matches the known top of the sector).

## Errors & Resolutions

1. **First charities.json was 27 MB** — too heavy. Switched from object-per-record to tabular `{headers, rows}` with bit-fields for the boolean flags, dropped to 10 MB. Financials moved to a separate ABN-keyed file loaded lazily.

2. **TypeScript build errors:** unused `expenses` var in drilldown.ts (removed) and tests/csv.test.ts couldn't find types for the `.mjs` pipeline module (excluded `tests/` from `tsc` include — Vitest handles its own type-checking).

3. **Preview server conflict on port 5200** — there was a stale "benrichardson-dev" preview server holding port 5200. Stopped it, but the launch.json in the worktree CWD was scoped to that other project (and the site lives outside the worktree at `sites/2026-06-24-au-charities/`, so the preview tool couldn't reach it). Fell back to `npx vite preview` in the project dir via Bash and curl-tested.

4. **Chrome MCP not connected** for the visual snapshot. Skipped visual screenshot; verified via curl + HTML title + bundle syntax check + tests.

5. **`gh repo create` failed: HTTP 401**. Same root cause as the previous three sites (au-pollution, au-ndis-watch, au-agedcare). Logged as `deploy_failed`. After `gh auth login`, run:
   ```
   cd /Users/benrichardson/Code/gh-site-factory/sites/2026-06-24-au-charities
   gh repo create ben-gy/au-charities --private --source=. --push
   gh api repos/ben-gy/au-charities/pages -X POST -f build_type=workflow
   gh api repos/ben-gy/au-charities/pages -X PUT -f cname="au-charities.benrichardson.dev"
   ```
   DNS is already in place; only the GitHub side is pending.

6. **`git push` for the registry/index update failed** — same auth issue. Commit `c2e5087 Register au-charities (build complete, deploy blocked by gh auth)` is local; runs after `gh auth login`.
