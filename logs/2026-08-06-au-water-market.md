# Build Log: Water Market
**Date:** 2026-08-06
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty, so a six-way validation workflow (Opus 5, one
agent per candidate, each required to actually *fetch* its source rather than
describe it) ran over six gaps in the 66-site catalogue: ACCC Scamwatch, ACCC
product recalls, Medicare bulk billing, ATSB aviation occurrences, NCVER
vocational education, and BOM water markets. A judge then picked one.

- **Won:** `au-water-market` — the only candidate whose source was proven
  fetchable *and* rich enough for 5+ genuinely distinct views.
- **Rejected:** `au-bulkbilling` (best data in the batch, but health.gov.au
  asserts Crown copyright with an explicit non-commercial prohibition and the
  only CC-BY mirrors stop at 2016); `au-aviation-safety` (extraction worked, but
  ATSB states publicly that it *limits* bulk export of the occurrence database —
  publisher intent, not licence, is the blocker; also structurally close to
  au-crashes); `au-scams` (proven, but only 18 months of history after a 2025
  taxonomy reset, state is the finest geography, and it would compete with the
  ACCC's own live dashboard rather than fill a gap); `au-vet` (ncver.edu.au is
  100% behind a Cloudflare managed challenge — CI would be blocked).
- **Logged as an expansion, not built:** the ACCC recall register was
  re-assessed and is materially more tractable than the 2026-08-02 note assumed
  (the paginated RSS feed IS a complete bulk spine back to 1986). That belongs
  on the existing `au-vehicles` EXPANSION_IDEAS entry.

## Site Details
- **Name:** Water Market
- **Repo:** ben-gy/au-water-market
- **Category:** economy (indexed as data-explorers)
- **Audience:** irrigators in the southern Murray-Darling Basin who buy and sell
  water each season, plus brokers, ag lenders and valuers, farm economists and
  Basin reporters
- **Stack:** Vanilla TypeScript + Vite 6, Leaflet 1.9 from npm, hand-rolled SVG
- **Data strategy:** pipeline, **monthly** cron (day 11, 04:41 UTC). The source
  rebuilds daily, but monthly is the fastest this factory permits and is the
  honest cadence anyway: agencies report in arrears, so the trailing six weeks
  are structurally incomplete.

## Data Sources
- `http://waterstorageapp.bom.gov.au/prod/MarketsData/Allocation_Trades.csv` — 125.8 MB, 482,376 rows
- `http://waterstorageapp.bom.gov.au/prod/MarketsData/Entitlements_Trades.csv` — 47.7 MB, 156,679 rows
- `http://waterstorageapp.bom.gov.au/prod/MarketsData/Entitlements_On_Issue.csv` — 118.9 MB, 578,376 rows
- `http://www.bom.gov.au/water/dashboards/config-service.php` — the only place the three URLs exist
- `patterns/geo/au-states.geojson` — ABS ASGS state boundaries
All BOM water market data CC BY 4.0, © Commonwealth of Australia.

## Architecture Decisions
- **Vanilla TS, not React.** Nine views on one hash-routed page; no component
  tree to justify a framework.
- **Build-time pipeline, not client fetch.** Not a preference: the data host
  serves no CORS headers and its HTTPS listener does not answer (curl exit 35).
- **Config-driven source discovery.** The three CSVs are linked from no HTML page
  anywhere on bom.gov.au — the Water Markets dashboard is a Tableau embed inside
  an AngularJS shell, and the file locations live only in `config-service.php`.
  The pipeline reads that config every run and fails loudly if a URL moves,
  rather than hard-coding paths that would one day 404 into "zero trades".
- **State-level map.** The MDBA Water Resource Plan Area polygons were fetched
  and tested (19 real CC BY 4.0 features, 13.9 MB): **0 of 482,376 rows join to
  them by name**. Rather than hand-author a 344-row crosswalk — inventing a join
  — the map uses `origin_state`, which joins exactly. Descope recorded in the
  About panel and the README.
- **Deliberate non-overlap with au-dams.** au-dams is BOM *storage levels*; this
  is the *market*. No storage figure appears anywhere here, and the temptation to
  use storage as a proxy for supply was resisted on purpose.

## Test Results
- Tests written: **148**, across parse (unit), analysis (pure logic), layout
  (positional geometry), overlays (the dismissal contract), gates (assertions
  against the real shipped JSON).
- Tests passed: 148. Failed: 0.
- Three failed during development and each was a real finding:
  1. A reconciliation gate fired on a 2 ML difference in 96 million — rounding
     each bucket before summing. Fixed to reconcile raw accumulators.
  2. Two gate thresholds were written against the *lease-contaminated*
     entitlement number and failed once the lease filter landed. They were
     measuring the artefact, not the finding, and were rewritten to assert the
     claim's **direction** rather than a magnitude.
  3. Two layout/analysis expectations were simply wrong about correct code
     (tick spacing, rolling-mean window) and were corrected.

## Build Status
- npm install: pass
- npm test: pass (148/148)
- npm run build: pass
- Local preview: pass
- Production URL: verified (see below)

## Adversarial review — 6 lenses, 39 findings raised
A second Opus 5 workflow attacked the finished site from six independent angles
(data accuracy, statistical honesty, licensing/claims, frontend correctness,
accessibility/mobile, pipeline robustness), each instructed to **refute** and to
recompute everything from the raw CSVs rather than trust the site's own JSON.

**Confirmed and fixed — wrong numbers that would have shipped:**

1. **Leases pooled with sales inflated the headline thesis.** The entitlement
   file mixes outright sales with limited-term leases. Leases are only 0.9% of
   rows but large and cheap per megalitre, so in a volume-weighted average they
   dominate: they dragged the 2019-20 price down 37% and turned a **+15%** rise
   across the drought years into **+71%**. Now filtered. *The reviewer's
   diagnosis was right but its mechanism was wrong* — its test treated
   `duration_of_lease == '0'` as a lease, which is 13,802 ordinary transfers and
   invents a fake 94% reporting break at 2016-17. Only `duration > 0` is a lease.
2. **Two mis-keyed rows were the "widest gap" headline.** Nov 2024 read +197.4%
   because two rows at $7,335 and $7,350/ML — against a $145 median that month —
   supplied 59.4% of every dollar in the zone. The file reaches $620,000/ML. A
   $2,000/ML ceiling (2× anything the market has plausibly reached; an allocation
   cannot rationally exceed the perpetual right) drops 349 rows from *price* while
   keeping their *volume*. Nov 2024 now reads +26.9%.
3. **Reporting gaps rendered as 0.00% liquidity.** WA filed 329 entitlement
   trades in 2024-25 and **exactly zero** in 2025-26; the NT's last was 2013 and
   the ACT's 2018. Divided by stock those became "0.00%" beside Victoria's 3%,
   under a subtitle calling it "the liquidity of the permanent market". Now
   **suppressed, not clamped**: five states get no rate and the view names them
   and says why.
4. **A front-page card read "12222% of the median ($1/ML)".** Token $1
   considerations (which BOM's own `price_per_ML` marks as 0) collapsed a
   zone-month median to $1.00. Filtered; worst dispersion reading fell from
   33,948% to 47.6× — and that survivor is a genuinely bimodal groundwater
   zone-month, now expressed as a dollar range rather than a percentage.
5. **The About panel understated its own zero-price exclusion.** Claimed
   "roughly 40%"; the true drag is **−70%** ($149.87 → $45.54). Corrected.
6. **The map said "buys/sells" over volume that is ~40% unpriced.** Relabelled to
   send/receive — the water moved, the money often did not — and disclosed.
7. **The seasonal clock's caption contradicted its own encoding** (said radius =
   volume; radius is the year). Rewritten.
8. **The skip link was broken by the hash router** — `#main` parsed as a view
   name and bounced every keyboard user to Overview. Now a button.
9. **Mobile chart text rendered at 3.3 CSS px.** Charts now scroll at a legible
   width and authored type is scaled so it lands near 10px. Measured 9.5–16px.
10. **Tooltips were mouse-only** — every number unreachable on a phone or by
    keyboard. Added pointerdown (touch/pen) and focusin paths.
11. Plus: duplicated panel titles, a `.axis text` selector that matched nothing
    (every axis label was unstyled at browser default), churn computed on the
    in-progress season, WCAG contrast on table headers (4.26:1 → 5.15:1), 15px
    glossary targets → 24px, a hand cursor on ~1,270 non-clickable marks, an
    undisclosed flow-diagram second cap, an undisclosed heatmap date cut, a
    "0.05%" comment that was 35× wrong (real: 1.76%), JSON-LD crediting the
    dataset to the site author rather than the Commonwealth, http:// attribution
    links, and 178 lines of dead `svgZoom.ts` that nothing imported.

**Refuted:** "There are no GitHub Actions workflows at all" — a race with the
agent's own read; both workflows exist and were verified.

**Raised but NOT fixed, recorded honestly:**
- SVG chart marks remain keyboard-inaccessible (no tabindex/role on ~3,100
  delegated click targets); `role="img"` on chart roots also makes per-mark
  aria-labels inert. The tooltip now has a focus path, but full keyboard parity
  for every mark is a larger refactor.
- The Choke's two lines are distinguished by colour alone and collapse under
  protanopia; the reliability ramp's palest two lines are under 3:1.
- Data tables still require horizontal scrolling at 375px.
- Churn counts internal/administrative transfers as ownership changes.
- The 6-week provisional window is a judgement, not a measured lag curve.

## Deployment
- Repo created: yes — https://github.com/ben-gy/au-water-market
- GitHub Pages enabled: yes (build_type=workflow)
- Cloudflare DNS CNAME created: yes (au-water-market → ben-gy.github.io)
- TLS certificate: approved
- Directory entry live on main AND served: yes (67 sites, au-water-market first)
- Workflow triggered: yes
- Data Pipeline on CI: **success on the first run** — proves the 292 MB
  plain-HTTP fetch and the config-service.php discovery work on a clean runner.

### GitHub Pages platform failure — three attempts needed
The `build` job passed every time (npm ci, 148 tests, vite build, artifact
upload). The `deploy` job failed **twice** with `actions/deploy-pages@v4` sitting
at `Current status: deployment_in_progress` for its full 10-minute timeout and
then `##[error]Timeout reached, aborting!`. Nothing in this repo caused it: the
gh-site-factory repo's own unrelated Pages build errored in the same two-minute
window and also only recovered after a manual rebuild request, so this was a
transient GitHub Pages incident. The third attempt succeeded. Recorded here
because a reader of the Actions history would otherwise see two red runs and
assume a defect in the site.

## Production verification (live URL, not localhost)
- `https://au-water-market.benrichardson.dev/` → **200**
- The served bundle is byte-identical to the locally verified one (`index-CxKzbe-l.js`)
- All shipped assets 200: 9 data JSONs, the ABS GeoJSON, og.png, robots.txt,
  sitemap.xml, favicon.svg, third-party-notices.txt, the IndexNow key file
- All 9 views render on production: no blank view, no NaN/undefined/Infinity in
  any view's text, 2,551 hover-tooltip marks on the densest view
- `read_console_messages(onlyErrors)` → **zero errors**
- **Real click** (Chrome MCP, genuine pointer events — not `element.click()`) on
  a zones table row opened the drill-down for "1A Greater Goulburn" with correct
  figures (14.76 million ML, $144.76/ML, 90,898 trades of which 57,642 priced,
  net −383.8 GL) and a rendered price history
- **Modal dismissal contract, all four exits, asserted by computed style AND
  DOM detachment**: ✕ ✓, scrim pointerdown ✓, Escape ✓, real touch sequence ✓
  (no reopen), zero leftovers, scroll lock released, no `[hidden]` leaks,
  `elementFromPoint` at viewport centre returns page content again
- **The Leaflet z-index test**: About modal opened *from the map view*, screenshot
  confirms it paints fully above the map. Modal z-index 2100 vs Leaflet's 1000,
  contained by `isolation: isolate` on `.map-frame`. 8 tiles loaded, 8 real ABS
  polygons rendered
- Mobile at 375px: **no page-level horizontal overflow on any of the 9 views**;
  smallest chart label now 9.5–16 CSS px (was 3.3px before the fix)
- IndexNow ping: HTTP 202 (site) / 200 (hub). lab.benrichardson.dev redeploy triggered.

## Errors & Resolutions
- **`gh repo create` name check** — verified free against 300 repos first.
- **Stale browser cache repeatedly masked fixes.** Three separate verification
  rounds screenshotted an old bundle and old JSON. Resolved by cache-busting
  reloads; worth remembering that a screenshot can lie about a rebuild.
- **The preview tool started the wrong dev server** (a sibling site's
  launch.json at the worktree root). Replaced with this site's config.
