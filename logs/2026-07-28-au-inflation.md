# Build Log: Cost of Living
**Date:** 2026-07-28
**Status:** deployed

## Idea Source
Researched (IDEAS.md was empty). Consumer-price/inflation was a top-searched "cost of
living" topic with clean, uncovered ABS data. Verified against the registry (60 sites, no
CPI/inflation entry) and `gh repo list ben-gy` (no `au-inflation`). Distinct from the
economy siblings (au-income = ATO earnings by postcode, au-trade = merchandise trade,
au-jobs = unemployment, au-insolvency = bankruptcies) and from au-visas/au-super. Validated
the ABS CPI + WPI SDMX endpoints live before committing: 158 expenditure classes back to
1948, both reconciliation approaches confirmed viable.

## Site Details
- **Name:** Cost of Living
- **Repo:** ben-gy/au-inflation
- **Category:** economy (index: data-explorers)
- **Audience:** anyone worried about the cost of living — renters, retirees, students,
  journalists — plus readers comparing cities or checking wages vs prices.
- **Stack:** Vanilla TypeScript + Vite + Vitest (no runtime dependencies)
- **Data strategy:** pipeline. CPI is published QUARTERLY, so the cron is quarterly
  (`'23 6 8 2,5,8,11 *'` — the month after each late-Jan/Apr/Jul/Oct release, staggered day 8
  + off-hour). Embeds the latest data quarter (2025-Q3), not a run timestamp → idempotent.

## Data Sources
- ABS Consumer Price Index, Australia — index numbers (MEASURE 1), annual % change (MEASURE 3)
  and contributions (MEASURE 4) for 158 expenditure classes, 8 capitals + weighted average,
  quarterly 1948-Q3→2025-Q3. `https://data.api.abs.gov.au/rest/data/ABS,CPI,1.1.0/` (SDMX CSV).
- ABS Wage Price Index, Australia — total hourly rates ex bonuses, all industries, seasonally
  adjusted, quarterly 1997→2026-Q1. `https://data.api.abs.gov.au/rest/data/ABS,WPI,1.2.0/`.
- The CPI INDEX code list (`CL_CPI_INDEX`) drives the group→subgroup→class hierarchy
  (parent + description per code) — built from the authoritative source, not hand-typed.

## Architecture Decisions
- **No map.** CPI is temporal/categorical; the 8 capitals aren't choropleth-worthy. Same
  deliberate no-map call as au-baby-names, noted honestly. Leaned on temporal + categorical
  richness (10 views) instead.
- **Reused the au-baby-names architecture** (view registry + shell, tooltip/glossary/drawer
  components, chart primitives, treemap + svgZoom patterns) with a new teal/amber "paper"
  palette. Hand-rolled all SVG; zero runtime dependencies.
- **Dense series storage.** Each class series is stored as a start quarter + a dense value
  array; cumulative change, rebasing and purchasing power are computed client-side, so the
  whole dataset is one 195 KB `classes.json`.
- **Analytical series separated** from the expenditure tree by reachability from All-groups,
  so Tradables/Goods/Services/Discretionary don't pollute the treemap or the risers/fallers.

## Reconciliation Gates (both hard-fail, on independently-published ABS columns)
1. Computed annual change `(index[t]/index[t-4]-1)×100` (MEASURE 1) == ABS-published annual
   (MEASURE 3) for All-groups at every quarter → **0 mismatches across 305 quarters**.
2. The 11 groups' contributions (MEASURE 4) sum to the All-groups index number (143.6) →
   **0 mismatches** (checked on the quarters where all groups carry a contribution).

## Test Results
- Tests written: 71 (parse, layout, data/format/colours, render, interactions)
- Tests passed: 71
- Tests failed: 0
- Coverage: RFC-4180/SDMX parser, quarter maths, hierarchy, both reconciliation gates
  (incl. a deliberately-garbled series and a wrong-sum failure case), positional layout
  (squarify + divergingLayout: in-bounds / no overlap / flush zero-axis), client series
  helpers, a headless jsdom render of all 10 views + the per-item drawer (leaf / group /
  no-cum2000) + the About modal against the REAL built data (no NaN/undefined/Infinity,
  treemap tiles in-bounds), and interaction wiring (bar-click opens item, group scope → 11
  rows, histogram click filters + reveals Clear, treemap drill, insight chips resolve).

## Build Status
- npm install: pass
- npm test: pass (71/71)
- npm run build: pass (tsc + vite; no third-party runtime code in the bundle)
- Local preview: pass (HTTP 200; data served)

## Deployment
- Repo created: yes (ben-gy/au-inflation)
- GitHub Pages enabled: yes (workflow build)
- Cloudflare DNS: CNAME au-inflation → ben-gy.github.io created (grey cloud)
- Deploy workflow: green on first run
- Data Pipeline workflow: green on CI (reconciliation passes on a fresh checkout)
- PR created: https://github.com/ben-gy/au-inflation/pull/1
- Live: custom domain returns HTTP 200 with the byte-identical bundle (index-BoGLongo.js)
  and all data + SEO assets; `https_enforced` still issuing at hand-over (should flip on its own).

## Errors & Resolutions
- Contribution slice (MEASURE 4) came back ~31 KB (ABS only publishes contributions for the
  most recent ~year, all codes) — below the collect minBytes; lowered the threshold and noted
  why in a comment. Gate 2 checks the quarters where all 11 groups are present.
- `tsc` build initially picked up the tests dir (a `.mjs` import with no types → TS7016);
  scoped `tsconfig.include` back to `["src"]` (vitest typechecks tests via esbuild).
- Added `src/vite-env.d.ts` for `import.meta.env` types.

## Limitation
Live in-browser real-click + true-375px verification could not be run: the in-app Browser
pane blocks localhost/benrichardson.dev by policy and no Claude-in-Chrome browser was
connected (`list_connected_browsers` → `[]`). Correctness rests on the headless render +
interaction tests against the byte-identical production `dist`. Re-run production real-click
and mobile-overflow verification once a browser surface is available, and confirm
`https_enforced` flipped to true.
