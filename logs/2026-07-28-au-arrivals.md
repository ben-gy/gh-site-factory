# Build Log: Overseas Arrivals
**Date:** 2026-07-28
**Status:** deployed

## Idea Source
IDEAS.md was empty, so researched a new topic. Swept the 60+ existing ben-gy repos and
the registry to avoid duplication, then probed the ABS SDMX dataflow catalogue for an
untapped, reliable series. Found `OAD_COUNTRY` and `OAD_REASON` — the arrivals side of
Overseas Arrivals and Departures — genuinely uncovered (au-visas is visa *stock*,
au-population is *permanent* migration; this is short-term *movement*). Confirmed the
data before committing: monthly 1975-07→2026-05 (611 months), 78 country codes, and the
signature COVID cliff (Apr 2020 = 2,250 visitors vs a Dec 2019 peak of 1,077,720).

## Site Details
- **Name:** Overseas Arrivals
- **Repo:** ben-gy/au-arrivals
- **Category:** tourism (index: data-explorers)
- **Audience:** Curious Australians, journalists, tourism/economics analysts
- **Stack:** Vanilla TypeScript + Vite + Vitest + Leaflet
- **Data strategy:** pipeline — monthly cron (`23 5 9 * *`). The ABS publishes OAD ~5
  weeks after each reference month; monthly is the fastest cadence the factory allows and
  it matches the source. Embeds the latest data month (2026-05), not a run timestamp.

## Data Sources
- ABS OAD_COUNTRY — visitor arrivals & resident returns by country of residence/destination (data.api.abs.gov.au SDMX)
- ABS OAD_REASON — by main reason for journey and intended length of stay (data.api.abs.gov.au SDMX)
- Natural Earth 1:50m Admin-0 map units — world boundaries (public domain)

## Architecture Decisions
- **Vanilla TS** (not React): a single-page, tab-switched explorer with one Leaflet map — no
  component tree or routing that would justify React. 216 KB JS / 65 KB gzip.
- **Two reconciliation gates** on independently-published ABS columns:
  1. EXACT cross-flow total: OAD_COUNTRY total == OAD_REASON total every month (same
     population, identically rounded). 0 mismatches / 611 (visitors), 0 / 227 (returns).
  2. TOLERANT continental: members + Other == each `TOn` subtotal, enforced from 2004.
     The ABS randomly rounds every count to the nearest 10 (perturbation), so exact sums are
     impossible; a real parse error still fails everywhere including modern months.
- **No seasonally-adjusted series** exist in OAD_COUNTRY, so the smooth recovery line is a
  client-side 12-month rolling mean of the original series (kept honest — original is the
  default and the seasonal swing is the point of the Seasonality view).
- **Map:** reused the au-trade world-map machinery (Natural Earth fetch, mapshaper simplify,
  ISO_A3_EH join, Leaflet choropleth) but built a fresh SACC-numeric→ISO3 map for the 57
  OAD country codes (au-trade uses ABS merch-trade alpha codes, a different scheme).

## Test Results
- Tests written: 48
- Tests passed: 48
- Tests failed: 0
- Coverage: RFC-4180/SDMX parser; continent-hierarchy builder incl. the NI/0000-leak
  regression; BOTH gates incl. deliberate failure cases (shifted cross-flow, 10× continental
  drift); positional histogram-bin layout (flush boundaries, counts sum, no double-count);
  svgZoom viewBox math; headless jsdom render of all 8 non-map views + drawer + About against
  the real built data asserting no NaN/undefined/Infinity.

## Build Status
- npm install: pass
- npm test: pass (48/48)
- npm run build: pass
- Local preview: pass

## Deployment
- Repo created: yes (ben-gy/au-arrivals)
- GitHub Pages enabled: yes (workflow build)
- Cloudflare DNS CNAME: created (au-arrivals → ben-gy.github.io)
- PR created: https://github.com/ben-gy/au-arrivals/pull/1
- Deploy workflow: green on first run; Data Pipeline: green on CI
- Live: HTTP 200 at custom domain; live bundle byte-identical to local dist; live meta
  serves latestMonth 2026-05, 57 countries, trough 2250, recovery 0.968. HTTPS cert still
  issuing at hand-over (https_enforced=false, HTTP already 200) — should flip within ~15 min.

## Errors & Resolutions
- **Continental gate failed on first aggregate run (313/346 gross failures).** Diagnosed: all
  failures fell in 1975–2003, none from 2003 on → genuine early-history incompleteness (ABS
  broke out fewer individual countries before ~2003), not a parse error. Restricted the gate's
  pass/fail decision to ≥2004 where the breakdown is complete.
- **6 residual modern failures (returns, TO9 in early 2004).** Root cause was a real bug in the
  hierarchy builder: `NI` and `0000` (national "not stated") trail `OT9` in the codelist with no
  following `TOn`, so the scan folded them into Sub-Saharan Africa's member sum. Fixed to admit
  only country codes and `OTn` residuals as members. Gate then passed cleanly.
- **Seasonality heatmap layout** — a `width:100%` table with fixed-width cells expanded the Year
  column and bunched the month cells to the right with dead whitespace. Fixed with
  `table-layout: fixed` + a fixed year column + `max-width` + centring.
- **Cosmetic:** a near-zero YoY rendered as "−0%" (fixed to "about the same as a year earlier");
  switching nav views now closes an open country drawer.
- **Limitation:** live-URL in-browser real-click verification could not run — the Browser pane
  blocks benrichardson.dev by policy and no external browser was connected. Verification rests on
  full real-click testing of the byte-identical local production build (matching bundle hash).
