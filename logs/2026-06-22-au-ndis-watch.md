# Build Log: NDIS Watch (AU)
**Date:** 2026-06-22 (built) · 2026-06-26 (deployed)
**Status:** deployed

> **Update 2026-06-26:** `gh` auth restored. Repo created (`ben-gy/au-ndis-watch`), pushed, GitHub Pages enabled (workflow build), Cloudflare DNS CNAME created, custom-domain CNAME set. First Deploy workflow run failed on a race (Pages not yet enabled when it started); re-running after Pages was enabled succeeded. Site is live and serving over the custom domain (title, JS/CSS assets, and all 3,203 actions' JSON verified via HTTP). PR opened at https://github.com/ben-gy/au-ndis-watch/pull/1. TLS cert was still provisioning at hand-off (HTTP 200; HTTPS pending Let's Encrypt issuance, normally <15 min) — a final CNAME cycle was triggered to push it along.

## Idea Source
Researched — IDEAS.md was empty. After ruling out ACECQA childcare quality ratings (data is behind Cloudflare and the snapshot XLSX is unguessable), I pivoted to the NDIS Commission Compliance Actions dataset on data.gov.au, which is openly downloadable as a 3 MB CSV with ~3,200 enforcement records spanning 2019–2026.

The data answers a question real people Google every day — "is this NDIS support worker / provider safe to let into my home?" — and is not already covered by any existing site in registry.json. The closest neighbour is `au-agedcare` (aged care quality ratings); NDIS enforcement is a fundamentally different domain (regulatory enforcement vs quality ratings) for a different audience (NDIS participants and families vs aged-care residents and families).

## Site Details
- **Name:** NDIS Watch (AU)
- **Repo:** ben-gy/au-ndis-watch *(not created — gh auth blocked)*
- **Category:** government-transparency
- **Audience:** NDIS participants and their families vetting providers/workers; plan managers and support coordinators; disability advocates and journalists; sector researchers
- **Stack:** vanilla TypeScript + Vite 6 + Leaflet 1.9
- **Data strategy:** pipeline (Node script fetches latest CSV from data.gov.au, parses, normalises, writes static JSON; refreshes weekly via GitHub Actions)

## Data Sources
- **NDIS Commission Compliance Actions** — https://data.gov.au/data/dataset/ndis-commission-provider-register-part-2
  - Format: CSV via CKAN API, weekly republication
  - Records: 3,203 (as of 2026-06-04 dataset)
  - Fields: Type, Date effective from, Date no longer in force, Name, ABN, City, State, Postcode, Provider Number, Registration Groups, Relevant information (narrative)
  - Six action types: Compliance notice (2,015), Banning Order (760), Revocation (330), Refusal to re-register (54), Suspension (39), Enforceable Undertaking (5)
- **ABS Estimated Resident Population (June 2024)** — embedded in `pipeline/lib/population.mjs`, used for per-100k state rates

## Architecture Decisions
- **Vanilla TypeScript over React.** Site has 7 views but is fundamentally a search-and-filter explorer over a single dataset. Vanilla keeps the bundle to 205 KB (gzip 61 KB) — well under the React tax for an SPA of this size.
- **Hand-rolled SVG charts.** Timeline (stacked bars), Types (donut + cross-reference matrix), Groups (Sankey-style flow), and States (paired bars) are all hand-built SVG with hover tooltips. No D3 or chart library — keeps the bundle small and the styling fully under our control.
- **Leaflet for the map** (200 KB) — the one library that's worth the weight. Renders state-level proportional circles at hand-coded centroids; clicking jumps to Browse with the state filter applied.
- **Pipeline over runtime fetch.** The CSV is 3 MB and only updates weekly. Fetching it at runtime would be wasteful — much better to parse it once in a GitHub Actions cron, write smaller JSON, and serve statically.
- **Custom RFC 4180 CSV parser.** The narrative field contains commas, quotes, and newlines — every edge case CSV libraries are notorious for getting wrong. We wrote a 60-line parser with 9 dedicated tests instead of pulling in a 50 KB dependency.
- **Severity-driven colour system.** Red (banning / revocation), Amber (suspension / refusal), Slate (compliance notice / undertaking). Every chart, table cell, map bubble, and pill uses the same three-colour vocabulary. Users can scan visually for "where is the serious stuff" without reading.
- **URL hash for state.** View, filters, search, and drill-down entity are all encoded in the URL fragment — every shareable. A journalist can copy `https://au-ndis-watch.benrichardson.dev/#view=states&type=banning_order` and share that exact slice.

## Test Results
- Tests written: 81 across 7 files
  - `format.test.ts` — 24 tests for number/date/HTML escaping/truncate/relative age helpers
  - `csv.test.ts` — 9 tests for the RFC 4180 parser (quoted commas, quoted newlines, escaped quotes, missing trailing newline, CRLF, missing columns)
  - `types.test.ts` — 13 tests for action-type normalisation and severity mapping
  - `groups.test.ts` — 10 tests for registration-group classification (numeric codes and text descriptors)
  - `filter.test.ts` — 10 tests for filter combinations, sort orders, and entity lookup
  - `state.test.ts` — 10 tests for URL hash parsing and round-tripping, active-filter counting, filter reset
  - `insights.test.ts` — 5 tests for auto-generated insights (recent bans, YoY change, repeat offenders, state outliers, empty actions array)
- **All 81 tests pass.**

Initial run had 1 failure: the state-outlier insight test failed because the median calculation picked the upper of two values when N=2. Fixed by using proper interpolated median (mean of two middle values when count is even).

## Build Status
- npm install: ✅ pass (116 packages, ~8s)
- npm test: ✅ pass (81/81)
- npm run build: ✅ pass after fixing two TS strict-mode issues:
  1. Added `vite/client` to tsconfig types so `import.meta.env.BASE_URL` typechecks
  2. Added `allowJs: true` + a `pipeline/lib/index.d.ts` declaring the .mjs module exports so the test files can import them
- Final bundle: `dist/assets/index-UGJno_Kn.js` 205.84 KB (gzip 61.53 KB) + `dist/assets/index-BStAy7vw.css` 38.74 KB (gzip 11.18 KB) + `dist/index.html` 1.31 KB
- Local preview at `http://localhost:5301/`: ✅ HTTP 200 on `/`, `/data/stats.json` (3203 actions parsed), `/data/actions.json` (3203 actions), `/assets/index-*.js`, `/assets/index-*.css`
- Visual inspection via computer-use/Chrome MCP: ⚠️ skipped (autonomous run; no interactive permission grant available). Text-based verification (HTTP status, asset references in HTML, data JSON shape) is sufficient given clean build + tests pass + the production bundle is a deterministic transform of the source.

## Deployment
- Repo created: ❌ failed — `gh repo create` returned `HTTP 401: Requires authentication`
- DNS / Pages / Cert / Index: ❌ blocked downstream of repo creation
- PR: ❌ blocked
- Local commit: ✅ `7d4b16f Initial commit: NDIS Watch (AU) — every NDIS Commission compliance action, searchable` on local `main` branch in `sites/2026-06-22-au-ndis-watch/`

`gh auth status` reports: `The token in default is invalid.` This is the same root cause that blocked the previous build (`au-agedcare` on 2026-06-21). The fix requires user intervention: `gh auth login -h github.com`.

**Recovery steps for the user once auth is restored:**
```bash
cd sites/2026-06-22-au-ndis-watch
gh repo create ben-gy/au-ndis-watch --private --source=. --push
gh api repos/ben-gy/au-ndis-watch/pages -X POST -f build_type=workflow
# DNS via Cloudflare CNAME au-ndis-watch → ben-gy.github.io
# cert cycle: gh api repos/ben-gy/au-ndis-watch/pages -X PUT -f cname="" && sleep 3 && gh api repos/ben-gy/au-ndis-watch/pages -X PUT -f cname="au-ndis-watch.benrichardson.dev"
```

## Errors & Resolutions
1. **ACECQA quality data is behind Cloudflare.** The childcare-quality idea fell apart when the official "Find a Service" CSV download URLs returned 404 outside Cloudflare protection, and the quarterly NQF Snapshot xlsx isn't published at a guessable path. Resolution: pivoted to the NDIS Commission Compliance Actions dataset on data.gov.au, which is openly downloadable.

2. **One test failed initially** — median calculation with N=2 in `buildInsights`. Resolution: switched to proper interpolated median when count is even.

3. **TS build failed with `import.meta.env` and .mjs imports.** Resolution: added `vite/client` to types; added `pipeline/lib/index.d.ts` ambient module declaration; flipped `allowJs: true` in tsconfig.

4. **Preview server port conflicts.** The `mcp__Claude_Preview__preview_start` wrapper kept trying port 5200 (held by another session) and ignored my `autoPort: true` configuration. Resolution: dropped to `npx vite preview --port 5301` via Bash, verified HTTP 200 and data-fetch endpoints directly.

5. **GitHub auth expired.** Same issue that blocked the previous build. Logged, marked deploy_failed, continued to registry/log updates per the spec.
