# Build Log: Imprisonment
**Date:** 2026-07-24
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty. The fleet is already dense with ABS-joined
geographic data explorers, so I looked for a genuinely new domain with clean,
free, authoritative data and a strong "the raw number hides the truth" angle.
Web research surfaced two strong candidates (product recalls; ABS Prisoners in
Australia). Recalls had no confirmed clean pipeline and no geography. **ABS
Prisoners in Australia** was chosen: national, free, annually published, and it
carries one of the most significant hidden-truth data stories in the country —
Aboriginal and Torres Strait Islander over-representation — that no existing
fleet site covers (au-crime is recorded *victims* of crime, not prisoners).

## Site Details
- **Name:** Imprisonment
- **Repo:** ben-gy/au-imprisonment
- **Category:** justice (index category: data-explorers)
- **Audience:** journalists, students, justice/community-sector workers, engaged citizens
- **Stack:** Vanilla TypeScript + Vite + Vitest
- **Data strategy:** pipeline, **annual cron** (`23 6 9 12 *`) — ABS publishes *Prisoners in Australia* each December, the slowest proportional cadence. Embeds the ABS release date, not a run timestamp, so re-runs are idempotent.

## Data Sources
- ABS Prisoners in Australia 2025 — Prisoner characteristics, Australia (Tables 1–14): national time series (T2), offence series (T5), offence×legal-status (T9), age×sex (T4).
- ABS Prisoners in Australia 2025 — States and territories (Tables 15–35): state series (T16), crude + age-standardised rates by Indigenous status (T18/T19), time on remand (T32), prison facilities (T34).
- ABS ASGS 2021 state & territory boundaries (patterns/geo/au-states.geojson).

## Architecture Decisions
- Vanilla TS over React — single-page tabbed explorer, no component tree or routing library needed.
- Adapted the au-branches app shell (proven sticky-header/clip, map isolate z-index, drawer/modal/tooltip stacking, overflow-safe grids) to a new civic light palette: indigo structure, ochre for the Aboriginal & Torres Strait Islander series (red-earth, respectful, clearly distinct from the muted slate of the non-Indigenous series), gold for remand.
- Charts hand-rolled SVG (dumbbell, stacked area, pyramid, histogram, matrix split, sparklines); Leaflet only for the choropleth; squarify/svgZoom/tooltip/feedback copied from patterns/.
- Signature view chosen for the data shape (few jurisdictions, deep categories, a decade of history): a ranked **dumbbell on a log axis** — the ratio *between* the two rates is the argument, and a log axis keeps a 130-per-100k dot and a 4,600-per-100k dot both legible. No network graph (nothing here is entity-to-entity).

## Data traps handled (in the dependency-free, unit-tested pipeline)
1. **Year-row over-run:** `yearOf` matched any 4 digits, so footnote paragraphs and the trailing "Australia" national block were absorbed into the last state (ACT), doubling the national total. Fixed with `rowYear` = *exactly* a 4-digit label, and any non-year/non-state row ends the current block.
2. **Cell perturbation:** the ABS randomly adjusts small cells (T4/T9/T34 footnotes), so component sums fall a few short of the published total. The HARD gate asserts the two *independently-published* national totals agree exactly (T5 offence == T2 characteristics == 46,998); component sums (state/offence×status/remand/facility) are tolerance-checked with warnings.
3. String sentinels (np, .., -, n.a.) → null, not 0; footnote markers stripped; state header labels (Vic./Qld/Tas./Aust.) mapped to codes; FP noise rounded.

## Test Results
- Tests written: 40 (across 4 files)
- Tests passed: 40
- Tests failed: 0
- Coverage: cell helpers + all 9 extractors + aggregate/reconciliation (parse.test.ts), squarified treemap positional correctness (layout.test.ts), median/rankBy/insights (analysis.test.ts), headless render of all 8 non-map views + drawer (views.smoke.test.ts). The smoke test caught two issues pre-deploy: attachSvgZoom reading `viewBox.baseVal` (undefined in jsdom → guarded), and an over-strict tooltip assertion on the insights view.

## Build Status
- npm install: pass
- npm test: pass (40/40)
- npm run build: pass (tsc strict + vite)
- Local preview: the in-app Browser pane and Claude-in-Chrome both cannot reach the local dev server in this environment; verified against the live production URL instead (bundle is byte-identical).

## Deployment
- Repo created: yes (ben-gy/au-imprisonment)
- GitHub Pages enabled: yes (workflow build); custom domain au-imprisonment.benrichardson.dev, Cloudflare CNAME created, cert cycled
- PR created: https://github.com/ben-gy/au-imprisonment/pull/1
- Workflow triggered: yes — Deploy succeeded; Data Pipeline also ran green on CI (confirms collect.mjs works on GitHub Actions)

## Production verification
Real clicks in Chrome against https://au-imprisonment.benrichardson.dev: gap dumbbell → WA drawer (above content); map polygon → NT drawer (elementFromPoint confirms it paints ABOVE Leaflet); About modal opened FROM the map view paints above the map's panes; offences + prisons treemap hover tooltips fire; pyramid/choropleth/dumbbell render on real geometry with no overlaps. All 9 views: no console errors, no NaN, no page-level horizontal overflow.

## Errors & Resolutions
- Reconciliation initially hard-failed on component sums — corrected to a published-total-vs-published-total hard gate plus perturbation-tolerant component checks (the ABS explicitly perturbs cells).
- Year-row over-run bug (see traps) caught by the state-sum reconciliation and fixed.
- attachSvgZoom threw under jsdom — guarded on `viewBox.baseVal`.
- **Limitation:** a true 375px mobile viewport could not be exercised — both browser surfaces are blocked for benrichardson.dev / render at a fixed 1200px logical width here. Mobile responsiveness rests on the responsive CSS inherited verbatim from au-branches (previously 375px-verified) plus the headless render tests. Worth a manual phone check.
