# Build Log: Baby Names
**Date:** 2026-07-28
**Status:** deployed (HTTPS cert issuing at hand-over)

## Idea Source
Researched. IDEAS.md was empty. Scanned the registry (60+ sites, heavily AU civic-data
explorers) for a genuinely distinct, highly-searchable, evergreen idea. Chose **baby
names** — fun, general-public, extremely searchable ("most popular baby names
Australia"), and unlike anything in the catalog (temporal name-frequency, not
geographic). Confirmed open data exists with real counts before committing:
- NSW BDM Popular Baby Names — one CSV, 1952–2025, `Rank,Name,Number,Gender,Year`.
- QLD BDM Top 100 Baby Names — per-year wide files 2006–2025 + one 1960–2005 long file,
  all resolvable via the data.qld CKAN API.
Rejected a broader multi-state build (VIC/WA/SA/etc. publish only short/rank-only lists,
no open count history) — logged as an expansion idea instead. NSW + QLD together are
~45% of Australian births and the fashions are national.

## Site Details
- **Name:** Baby Names
- **Repo:** ben-gy/au-baby-names
- **Category:** data-explorers
- **Audience:** expecting parents, the name-curious, nostalgia browsers, journalists
- **Stack:** Vanilla TypeScript + Vite + Vitest, hand-rolled SVG (no chart/map libraries)
- **Data strategy:** pipeline, annual cron (`23 5 9 3 *`) — both registries publish the
  prior year each Jan–Mar; annual is the slowest correct proportional cadence. Embeds
  the latest data year, not a run timestamp, so re-runs are idempotent.

## Data Sources
- NSW Registry of BDM — Popular Baby Names 1952–2025 (data.nsw.gov.au, single CSV, counts).
- QLD Registry of BDM — Top 100 Baby Names 1960–2025 (data.qld.gov.au CKAN).

## Architecture Decisions
- **No map.** The data is name×year×sex — temporal ranking, not geographic. State is a
  2-value dimension (handled by the NSW-vs-QLD face-off). A Leaflet map would be
  pointless, so the signature form is a **bump chart** — the canonical way to show how a
  ranking evolves. This sidesteps all map-geometry rules cleanly.
- **Precompute in the pipeline.** meta.json (headline), names.json (per-name series +
  summary, ~495 KB), aggregates.json (diversity/first-letter/unisex/insights). Movers and
  the bump set are derived on the client from names.json. Total data ~550 KB, no libs →
  62 KB JS bundle.
- **Reconciliation gate on source ordering.** Three source shapes normalise to one record
  stream; the gate asserts count is non-increasing along each source's own ordering
  (NSW Rank, QLD row order) per (state,year,sex) group. A swapped Name/Number column
  breaks monotonicity, so 0 violations proves a correct parse. 0/188 ordered groups.

## Test Results
- Tests written: 47 (across parse, analysis, layout, format, render)
- Tests passed: 47
- Tests failed: 0
- One expectation was initially wrong (flash-in-the-pan threshold: Noah has exactly 2
  appearances so it qualified at maxYears:2) — fixed the test, not the code.
- Includes a headless jsdom render of all 10 views + the per-name drawer against the REAL
  built data (asserting no NaN/undefined and that charts produce marks), and a positional
  bump-layout test (in-bounds, finite, chronological x, rank 1 at top).

## Build Status
- npm install: pass
- npm test: pass (47/47)
- npm run build: pass (tsc + vite; 62 KB JS, 22 KB CSS, no third-party runtime code)
- Local preview: pass — full real-click verification in the in-app Browser pane against
  the byte-identical dist.

## Deployment
- Repo created: yes (ben-gy/au-baby-names)
- GitHub Pages enabled: yes (workflow build); Deploy workflow green on first run
- Cloudflare DNS: yes (CNAME au-baby-names → ben-gy.github.io)
- PR created: https://github.com/ben-gy/au-baby-names/pull/1
- Production: HTTP 200, live bundle index-D8yDkvMu.js matches local dist byte-for-byte,
  data + og all 200. HTTPS cert still issuing at hand-over (https_enforced=false) — should
  flip within ~15 min; re-check and re-verify over HTTPS.
- Registry + index updated on main; IndexNow pinged (202/200); lab hub redeploy triggered.

## Verification (local, against byte-identical dist)
- Overview: John→Noah hero, #1 cards, 39%→23% top-10-share stats, top-10 lists — all correct.
- Name Finder: John declines from #1 while Noah/Charlotte rise from outside the top 100
  (~1998) — the killer searchable feature works; end-labels disabled to avoid collision.
- The Charts: bump chart draws lines crossing across 74 years, end labels, zoom/pan.
- Diversity: four falling concentration lines + #1-count lines.
- First Letters: warm heatmap, J-names (1950s) → A/E names (2020s).
- Explorer: first-appearance histogram + sortable table (Michael #1, 98,464); copy tuned
  to match the roughly-steady histogram after the 1952 base bar.
- NSW vs QLD: side-by-side top lists + skew (genuine differences: Sienna/Chloe vs
  Matilda/Violet); rank bars confirmed rose #c9527e via computed style.
- Per-name drawer: REAL click on the Noah row opened it above the overlay (best #1 in 2025,
  21,008 babies, rank/count trajectory).
- Zero console errors; zero horizontal overflow at 375px across all 10 views + open drawer
  (asserted programmatically).

## Errors & Resolutions
- Two initial TS build errors (unused `current` in drilldown; `import.meta.env` typing in
  main) — removed both.
- Made `attachSvgZoom` return an inert handle when `viewBox.baseVal` is unavailable, so the
  headless render test can exercise the bump view under jsdom.
- Minor copy/polish after visual review (finder end-label collision; Explorer histogram
  wording) — fixed and rebuilt.
