# Build Log: Threatened Species
**Date:** 2026-07-21
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty. Scanned the registry + `gh repo list ben-gy` and found the environment category held only industrial pollution (au-pollution) — no biodiversity/threatened-species/conservation site. WebSearch surfaced the data.gov.au "Threatened Species and Ecological Communities of National Environmental Significance" dataset (EPBC Act state lists, DCCEEW/SPRAT); confirmed it's a free, no-auth CSV with per-species status, taxonomy and per-jurisdiction occurrence. Genuinely new domain, highly searchable, map- and hierarchy-friendly.

## Site Details
- **Name:** Threatened Species
- **Repo:** ben-gy/au-threatened
- **Category:** environment (index: data-explorers)
- **Audience:** general public, students/teachers, naturalists, journalists — low domain knowledge, phone-first, every term has an inline explainer
- **Stack:** Vanilla TypeScript + Vite + Vitest, Leaflet
- **Data strategy:** pipeline — quarterly cron (`23 4 9 1,4,7,10 *`). Source is amended a few times a year; quarterly is proportional and within the monthly-fastest rule.

## Data Sources
- EPBC Act Threatened Species State Lists — DCCEEW/SPRAT, via data.gov.au (dataset `threatened-species-state-lists`, resource `20260206spcs.csv`, 2,208 species). https://data.gov.au/data/dataset/threatened-species-state-lists
- ABS ASGS 2021 state & territory boundaries — reused `patterns/geo/au-states.geojson`.

## Architecture Decisions
- Vanilla TS (single-page, hash-routed tabs) — no need for React; smaller bundle (220 KB / 68 KB gzip).
- Pipeline with a dependency-free RFC-4180 parser in `pipeline/parse.mjs`, imported by the test suite (CI-traps rule). `collect.mjs` fetches the CSV with a browser UA (data.gov.au 403s bare clients — verified node fetch works before building), builds the compact `species.json`, and asserts status/kingdom sub-totals reconcile before writing.
- No time-series view: the source carries no per-species listing date, so a timeline would be fabricated — deliberately omitted. No force-graph: species don't connect, so a co-occurrence network would be a structureless dot-cloud (discouraged); the treemap + Class×jurisdiction matrix carry the relational load instead.
- Copied patterns: `tooltip.ts`, `treemap.ts`→`squarify.ts`, `svgZoom.ts`, `feedback.ts`, plus the positional layout-test template.

## Test Results
- Tests written: 43 (across 4 files)
- Tests passed: 43
- Tests failed: 0
- Coverage: `parseCsvRows`/`parseCsv` (quoted commas, doubled quotes, CRLF/BOM), `buildDataset` (status mapping, juris extraction, endemic counts, '(no class)' cleaning, '-' handling, total reconciliation), aggregation helpers (`groupBy`/`jurisCounts`/`endemicCounts`/`spanHistogram`), `zoomViewBox`/`clampViewBox` math, `computeInsights` (determinism + content), and positional treemap layout (in-bounds, no-overlap, no-NaN, area conservation).
- Initial `tsc` failures: unused imports (StatusCode/esc/Species) and tests being type-checked against the untyped `.mjs`. Fixed by dropping the imports and narrowing tsconfig `include` to `["src"]` (vitest type-checks nothing, runs fine).

## Build Status
- npm install: pass
- npm test: pass (43/43)
- npm run build: pass (dist/assets/index-g8ORQ5ee.js)
- Local preview: pass (dev server, all 8 views verified with real clicks)

## Deployment
- Repo created: yes (ben-gy/au-threatened)
- GitHub Pages enabled: yes (workflow build)
- DNS: Cloudflare CNAME au-threatened → ben-gy.github.io created; TLS cert cycled, `https_enforced=true`
- PR created: https://github.com/ben-gy/au-threatened/pull/1
- Workflow triggered: yes (Deploy succeeded; Data Pipeline also ran green)

## Verification (production, HTTPS)
- `https://au-threatened.benrichardson.dev` → 200; live bundle filename matches local dist byte-for-byte; og.png/robots.txt/sitemap.xml/favicon.svg + data/species.json all 200.
- All 8 views render on production; zero horizontal overflow at 375px on every view and with a drawer open; no console errors.
- Real-click interaction confirmed: treemap kingdom-tile drill (→ 11 class tiles), Explorer row → species drawer (#s=66657, full taxonomy chain + SPRAT link), map polygon → WA state drawer (614 listed / 482 endemic). About modal and both drawers paint fully above the Leaflet map (z-index isolate + 2000+).
- Two polish bugs found and fixed in-browser before deploy: (1) the global hover-tooltip didn't clear on view change (added `hideTooltip()` on render); (2) a drawer opened via deep-link/hash persisted across plain-view navigation (route() now closes it when the hash has no `#s=`/`#state=`). Also suppressed Leaflet's default focus bounding-box on click.

## Errors & Resolutions
- data.gov.au 403s bare fetch clients → pipeline sends a browser User-Agent + Accept header (verified against node fetch before building).
- The site was scaffolded under the main repo path, not the worktree, so `preview_start` needed an absolute `--prefix` in `.claude/launch.json`.
- The quarterly pipeline stamps `generated` (ISO now) into species.json, so it commits a fresh file each run even when data is unchanged — noted in the registry; harmless at quarterly cadence.
