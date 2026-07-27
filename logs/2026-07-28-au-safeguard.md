# Build Log: Safeguard Emitters
**Date:** 2026-07-28
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty. Scanned the 58-site registry (heavily Australian government-data explorers) for an
absent major domain and found **greenhouse-gas / carbon emissions** was uncovered — `au-pollution` is NPI *toxics*
(local pollutants), `au-solar` is rooftop generation, `nemwatch` is live electricity prices, but nothing covered carbon.
WebSearch confirmed the Clean Energy Regulator had freshly published the **2024–25 Safeguard baselines and emissions**
data (CSV, 54 KB). Verified it is public, no-auth, browser-fetchable, and distinct from any existing site (different
regulatory framework, different facilities, different substances, different story).

## Site Details
- **Name:** Safeguard Emitters
- **Repo:** ben-gy/au-safeguard
- **Category:** environment (index: data-explorers, country AU)
- **Audience:** Climate-policy analysts, journalists, investors, engaged citizens
- **Stack:** Vanilla TypeScript + Vite + Vitest
- **Data strategy:** pipeline — yearly cron (`23 6 9 5 *`, ~9 May) matching the CER's annual publication cycle; the
  slowest proportional cadence. Embeds the reporting-period end (2025-06-30), not a run timestamp, so re-runs are
  idempotent.

## Data Sources
- Clean Energy Regulator — Safeguard baselines and emissions data, 2024-25 (`cer.gov.au/document/baselines-and-emissions-table-2024-25`) and 2023-24 (`.../2023-24-baselines-and-emissions-table`).
- ABS ASGS 2021 state & territory boundaries (vendored into the repo at `pipeline/au-states.geojson`, from `patterns/geo`).

## Architecture Decisions
- **Copied the proven `au-vehicles` scaffold** and rewrote all domain modules (types, data, sectors, glossary, analysis,
  charts, about, drilldown, all 9 views, pipeline) — reusing the fleet infrastructure (main shell, tooltip, glossary
  popover, squarify, svgZoom, drawer-above-Leaflet, deploy/licensing files) rather than re-rolling patterns.
- **Vanilla TS** (no React) — matches the fleet; a single-page multi-view explorer needs no component framework.
- **State choropleth (8 states)** not a facility point-map — the data's geography is `State/Territory of operation`, and
  227 facilities as points would be sparse; the per-state panel + facility drawer gives the drill-down.
- **Two hard reconciliation gates** exploit that the CSV publishes redundant columns: gas breakdown sums to covered
  emissions, and net position equals net emissions minus baseline — both independently, so agreement proves the parse.
- **Vendored the state GeoJSON** into `pipeline/` because the deployed repo is standalone (no `patterns/` dir) — the CI
  Data Pipeline would otherwise fail to find the boundaries.

## Per-view UX critique (recorded per factory rule)
- **Overview** — Q: how bad is it? Hero "148 of 228 over the line" + gas split. Hover: exact Mt on every segment. Click: top-10 bars open facilities, sector bars jump to Sectors.
- **Over the Line** (signature) — Q: is big the same as over? No — log-log scatter with the at-baseline diagonal shows it. Hover: per-facility figures. Click a dot: facility drawer. Drag pans, wheel zooms (attachSvgZoom).
- **Map** — Q: where are the emitters? State choropleth (4 metrics incl. a diverging over/under ramp). Click a state: its facility list; click a facility: drawer over Leaflet.
- **Sectors** — Q: which industries? Treemap by covered emissions; click a tile: re-ranks + facility list; every cell has a hover tip.
- **Companies** — Q: who owns the emissions? Parent-company rollup bars + table; click: facilities.
- **Carbon Market** — Q: earned vs bought? Two-sided credit bars + per-sector table + earner/surrenderer leaderboards.
- **Gases** — Q: is it all CO2? No — 100%-stacked gas mix per industry (coal = 61% methane) + methane-intensity bars.
- **Explorer** — Q: find/compare any facility? Search + state/sector/position filters + sortable table + covered/baseline histogram with click-to-filter.
- **Insights** — auto-detected cards with facility chips (open drawer).
No dead clicks: every cell/bar/row/chip either opens a drawer, selects, or filters.

## Test Results
- Tests written: 51 (across 5 files)
- Tests passed: 51
- Tests failed: 0
- Coverage: RFC-4180 parser + sentinel/state/sector/company canonicalisers; BOTH reconciliation gates against the real
  built `facilities.json`; analysis rollups (bySector/byCompany/byState), rankedBy, histogram, buildInsights; positional
  squarified-treemap layout (bounds, no-overlap, area); formatters + svgZoom viewBox math; and a headless jsdom render of
  every non-map view + facility drawer + About modal against the real data asserting no NaN/undefined/Infinity.
- Two jsdom-only failures fixed during the run: `attachSvgZoom` now guards a missing `SVGSVGElement.viewBox` (jsdom/SSR
  safe), and the render test accepts text-only Insights cards.

## Build Status
- npm install: pass
- npm test: pass (51/51)
- npm run build: pass (tsc + vite, clean)
- Local preview: pass (real-click through all 9 views)

## Deployment
- Repo created: yes (ben-gy/au-safeguard)
- GitHub Pages enabled: yes (workflow build)
- DNS: Cloudflare CNAME au-safeguard -> ben-gy.github.io created; Pages CNAME set + cycled for cert
- Custom domain: https://au-safeguard.benrichardson.dev returns 200 over a valid trusted cert (https_enforced flag still
  flipping at hand-over; TLS already live)
- PR created: https://github.com/ben-gy/au-safeguard/pull/1
- Deploy workflow: green on first run
- Data Pipeline workflow: green on first run (validates the vendored geojson + reconciliation gates work on CI)
- Live verification: real production click opened the Gorgon facility drawer ABOVE Leaflet with correct figures; zero
  console errors; live bundle byte-identical to local dist.

## Errors & Resolutions
- **Stale `.claude/launch.json`** in the worktree pointed the preview at au-foreign-owned — rewrote it for au-safeguard.
- **jsdom viewBox / insights render** test failures — fixed as above.
- **Pipeline geojson path** — the collect script sourced boundaries from `../../patterns/`, absent in the standalone
  deployed repo; vendored `pipeline/au-states.geojson` and made collect prefer it (patterns dir is now only a local-dev
  fallback). Confirmed by the green CI Data Pipeline run.
