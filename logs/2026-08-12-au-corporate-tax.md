# Build Log: Corporate Tax Transparency
**Date:** 2026-08-12
**Status:** deployed

## Idea Source
Researched. `IDEAS.md` was empty, so an eight-way parallel research workflow (Opus 5, ultracode)
probed candidate domains not covered by the fleet, each agent required to **prove** data access
by fetching the actual machine-readable file and checking the body — not the landing page.

Two of eight died on access, exactly as intended:
- **ACIC National Wastewater Drug Monitoring** — PDF-only, end to end. Nine files per release, all
  PDF; nothing on data.gov.au (CKAN count 0); the AIHW mirror has no spreadsheet. The per-region
  series are chart art. Rejected.
- **NCVER VET statistics** — rejected on access.

Six were viable. Three judge panels (utility / data-quality / visualisation lenses) returned three
*different* winners, so the call was made in the main loop: **ATO Corporate Tax Transparency**, on
the grounds of a genuinely new domain for the fleet, ~6,000 searchable company names (the strongest
long-tail SEO surface of any candidate), the lowest data-access risk of the six, and a guaranteed
annual news cycle. `cancer-screening-incidence` was rejected despite winning the visualisation lens
because a participation-rate-by-PHN choropleth is structurally the same product as the existing
`au-immunisation`.

## Site Details
- **Name:** Corporate Tax Transparency
- **Repo:** ben-gy/au-corporate-tax
- **Category:** government transparency
- **Audience:** journalists and researchers on desktop; and, in far larger numbers, people arriving
  on a phone from a headline wanting to look up one company by name
- **Stack:** Vanilla TypeScript + Vite 6, no framework, no charting library, no runtime dependencies
- **Data strategy:** pipeline, **yearly** cron (`41 19 8 11 *`, 8 November) — the ATO publishes once
  a year in late October/November, so anything faster is pure churn

## Data Sources
- ATO **Corporate Tax Transparency — Report of Entity Tax Information**, 2013-14 to 2023-24, via
  data.gov.au CKAN: `https://data.gov.au/data/api/3/action/package_show?id=corporate-transparency`
  (11 XLSX workbooks, CC BY 3.0 AU, no key, no WAF)
- ATO **Corporate tax transparency report** commentary pages on ato.gov.au — used only as external
  reconciliation anchors. Blocks plain fetches; needs a browser UA plus `Sec-Fetch-*` headers.

## Architecture Decisions
- **Vanilla TS.** Seven hash-routed views, one shared drill-down drawer, no routing library needed.
  Final bundle 96 KB (30 KB gzipped) with **zero third-party code in it** — verified against the
  `sources` list in the sourcemap rather than `npm ls`.
- **`pipeline/parse.mjs` is dependency-free**, so the 44 parser tests run on CI without installing
  the pipeline package. All XLSX I/O lives in `collect.mjs`.
- **Everything derives from one payload in the browser.** `panel.json` (1.26 MB, 28,661 rows) is
  read by every view, so the treemap, the matrix and the drawer cannot disagree with each other.
  A test asserts `meta.totals` recomputes exactly from `panel.json`.
- **No map.** The data has no geography at all — the first fleet site with none — so the weight
  falls on the panel, funnel, streak-grid, treemap, bump and scatter views.

## Parsing traps hit and handled
1. **The triple-count trap.** The 2013-14 workbook has three overlapping data sheets — December
   (1,561), March (321) and Combined (1,859, the deduplicated union). Reading all three reads the
   first year roughly twice and invents a collapse in entity numbers the following year. A gate
   asserts the guard is *load-bearing*: reading every sheet must yield materially more rows (3,741)
   than reading only Combined (1,859).
2. **The phantom mega-entity.** 240 rows across eleven years have no ABN. Keying on ABN alone folds
   all of them into one fabricated entity present in all eleven years, near the top of the
   never-paid ranking. Blank ABNs fall back to a name key; a gate asserts the count cannot collapse
   below rows ÷ years.
3. **Four sheet names, two header positions.** `Combined`, `2014-15`, `Income tax`,
   `Income tax details`; header on row 0 or row 1 under a title. Solved by searching for the row
   that looks like a header rather than a per-year offset table.
4. **My own guard ate a real company.** The skip-the-blurb rule was `/^note[:s]?\b/i`, which
   silently deleted **NOTE PRINTING AUSTRALIA LIMITED** — the RBA's banknote printer — from every
   year, putting the entity count exactly one below the ATO's published figure. The colon is now
   required, and a test names the company.
5. **Prior-year rows.** 552 rows arrive in a later workbook tagged with an earlier income year. The
   `Income year` column is authoritative over the filename; the 2013-14 file has no such column.

## Reconciliation gates: 14/14 passing
Anchored against the ATO's own published commentary for **two** years, because one anchor cannot
distinguish "the parser is wrong this year" from "the source behaves like this every year":
- **2023-24 exact:** 4,110 entities / $3,278.8b income / $95.7b tax payable — matches to the
  published rounding on all three.
- **2022-23 directional:** the panel must *exceed* the as-published 3,985 / $3,138.4b / $97.9b by a
  bounded amount, proving late lodgers are routed backwards without double-counting. "Within 3%"
  alone would also pass a panel that silently dropped every one of them.

**The taxable-income divergence.** Summing the ATO's own published taxable-income column gives
6-8% *more* than the ATO's stated aggregate ($394.2b vs $365.5b in 2023-24), while total income and
tax payable reconcile exactly. My first explanation was wrong; an adversarial verifier found the
real one on the ATO's own page: the law only requires publication of a taxable income where it is a
positive amount, so losses appear as blanks and the column can only sum upwards, while the ATO's
aggregate is net of losses — which is why its industry table can show a whole segment as negative.
The gate now fails if the gap **disappears or grows**, because either means the source changed and
the site's on-screen explanation is stale.

## Verification: what the adversarial pass found
A second workflow ran nine "refute this claim" agents plus four sweep lenses, all Opus 5, each told
to break the site rather than confirm it. Six of nine claims were refuted and every finding was
real. All were fixed:

- **BLOCKER, entity-vs-group.** The Movers view's stat tiles read "Largest payer 2013-14: BHP →
  2023-24: Rio Tinto", implying Rio overtook BHP. False at group level: BHP lodges under two
  entities ($6.01b + $2.11b = $8.12b) against Rio's $6.25b. Tiles relabelled "largest single
  entity", and the view now computes and prints the worked example from the data.
- **BLOCKER, defamation risk ×2.** The drawer's lawful-explanation caveat was gated on
  `neverPaid`, so a company that paid once and reported nil for nine years got the raw figures with
  no caveat at all. And both caveats offered only loss-based explanations, which cannot explain the
  95 of 236 streak companies that reported *positive* taxable income and still owed nil — only
  offsets can. The caveat now triggers on the pattern and branches on which case applies.
- **Wrong tax fact.** The glossary said subsidiaries in a tax consolidated group "report nil". Under
  the single entity rule they stop lodging entirely and drop *out* of the report. Rewritten.
- **Self-contradiction.** The Sankey's tooltip printed "2.9% of total income" against the tax node —
  the exact ratio the site tells the reader is meaningless and promises never to show. Now per-node.
- **Nil defined two ways.** The Rates view's nil band was `no rate OR no tax`, which put Zurich
  (which paid $9.8m) under a "Nil tax payable" legend and printed 1,137 where every other view said
  1,136. Now strictly nil-tax-payable, with a separate marked band for the one company that paid
  tax with no taxable income.
- **Silent truncation.** The streak count printed the 250-row render cap as if it were the finding.
- **Sector misclassifications.** MITSUBISHI DEVELOPMENT (a top-30 taxpayer, $7.1b) sat in "Property
  & construction" because `DEVELOPMENT` fired first — 38% of that whole sector. Also Paddy Power
  → utilities (`POWER`) and The Star → media (`ENTERTAINMENT`). All pinned by ABN.
- **Accessibility.** Glossary triggers were spans (the entire explanatory layer unreachable by
  keyboard); sortable headers were bare `th`; streak cells were spans holding the only copy of the
  per-year figures; the drawer had no accessible name; `role="tabpanel"` on `<main>` destroyed the
  main landmark; six sector colours and two text tokens failed WCAG AA. All fixed, all locked in by
  tests.

Also found and fixed by inspection: a regex bug where wrapping the alternation in `\b(...)\b` made
every truncated stem (`TECHNOLOG`, `MANUFACTUR`, `PROPERT`, `LOGISTIC`) unmatchable — roughly half
the classifier rules were dead. Fixing it moved sector coverage from 64.6% to 84.8% of tax payable.
And the Sectors matrix used a single global colour scale, which rendered every row but mining as an
identical pale wash; it now scales per row, which is the question the view actually asks.

## Test Results
- Tests written: **135** across 5 files (parse 44, hygiene 43, data 19, format 16, layout 13)
- Tests passed: 135 · failed: 0
- Four failures during development were my own new tests catching real bugs — the note-row guard,
  an over-strict gate on a fixture that lacked blank ABNs, a `--text-tertiary` at 4.48:1 (just under
  AA), and a hygiene rule matching its own explanatory comment. All fixed at the source.

## Build Status
- npm install: pass · npm test: pass (135/135) · npm run build: pass · local preview: pass

## Deployment
- Repo created: yes · GitHub Pages enabled: yes · DNS (Cloudflare CNAME): yes
- TLS certificate: **approved** — note `https_enforced` is `false` on this healthy site, which is
  exactly why the cert poll reads `.https_certificate.state` instead
- Workflow triggered: yes — deploy succeeded
- **The data-pipeline deploy dispatch worked end-to-end on the first run.** The pipeline ran on
  push, regenerated the data, committed `c09bd1f`, and explicitly dispatched `deploy.yml`; the live
  site now serves that commit. This is the fleet-wide defect where a `GITHUB_TOKEN` push goes green
  and never deploys — confirmed fixed here.
- Directory entry live on main: yes

## Production verification
- `https://au-corporate-tax.benrichardson.dev/` → HTTP 200, TLS verified (`ssl_verify=0`)
- The live JS, CSS, `index.html`, `panel.json` and `prrt.json` are **byte-identical** (SHA-256) to
  the local `dist/` that was real-click verified. `meta.json` differs in exactly one key —
  `generated` — because CI regenerated it; gates and totals are identical, 14/14 passing live.
- All eight static assets return 200. `ben-gy.github.io` 301s to the custom domain.
- **Not verified in a live browser.** Chrome MCP could not load the new external origin: this is a
  non-interactive scheduled run and the per-origin approval card cannot be granted, so `navigate`
  timed out three times. Every interactive check below was therefore performed against
  `vite preview` of the byte-identical production bundle, not against the production URL.

### What was verified with real clicks (local preview of the shipped bundle)
- All seven views render; zero console errors.
- **Zero horizontal overflow at 375px on every one of the seven views** (asserted, not eyeballed:
  `scrollWidth === clientWidth === 375` throughout).
- **The four-exit modal contract, at 375px, asserted by computed style** — About modal and
  drill-down drawer each closed by ✕, by scrim `pointerdown` (mouse *and* `pointerType:'touch'`),
  and by Escape, re-opening between each. Baseline clean (no `[hidden]` element leaking, viewport
  centre returns page content). Panels are detached from the DOM when closed, close button is a
  real 44×44 inside the panel, scroll lock releases.
- Real trusted clicks: nav tabs, a table row, and a treemap tile — each opening the drawer over the
  chart it overlaps, with the correct company and no stale tooltip.
- Screenshots of every view inspected individually.

## Errors & Resolutions
- **`XLSX.readFile` cannot read a relative path in this sandbox** — switched to
  `XLSX.read(fs.readFileSync(...))`.
- **First real click appeared dead** — it was a stale screenshot-coordinate cache on my side, not a
  site defect. The screenshot is 800×450 for a 1280×720 viewport (scale 1.6) and every click needs a
  fresh screenshot first. Confirmed working by computing the coordinate from `getBoundingClientRect`.
- **A stale tooltip hung over the opened drawer** — the hovered element never receives `mouseout`
  when an overlay appears under a stationary pointer. `openOverlay` now calls `hideTooltip()`.
- **Streak grid stranded names ~1,200px from their cells** on a wide screen; grid capped at 760px.
- **Two blank screenshots mid-run** were a Browser-pane capture artefact, confirmed by reading the
  DOM geometry and `elementFromPoint` — not a rendering fault.

## Notes for next time
- The ATO commentary pages are the only external anchor available and they must be transcribed by
  hand into `ATO_ANCHORS` in `collect.mjs` with each release. Add 2024-25 when it lands (~Nov 2026).
- Sector classification is a name heuristic with ~100 ABN overrides. A sampled error rate of roughly
  2% (uniform) to 13% (stratified) is the honest figure; `CONTRIBUTING.md` invites corrections.
- The `verify-corporate-tax-claims` workflow script is reusable for any future site making numeric
  claims — the "refute, don't confirm" framing found six real defects that inspection had missed.
