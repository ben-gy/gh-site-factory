# Build Log: Donations AU
**Date:** 2026-04-16
**Status:** deployed

## Idea Source
From IDEAS.md (first entry, now removed from queue):

> Australian political donations explorer — unified searchable interface across the AEC federal register and all 6 state donation registers (different thresholds, disclosure timelines, and formats). Search by donor, recipient, amount, date, and state.

## Site Details
- **Name:** Donations AU
- **Repo:** [ben-gy/au-donations](https://github.com/ben-gy/au-donations)
- **URL:** https://au-donations.benrichardson.dev
- **PR:** ben-gy/au-donations#1
- **Category:** government-transparency
- **Audience:** Journalists, policy researchers, electoral-law students, civic-minded citizens
- **Stack:** Vanilla TypeScript + Vite 6 + Vitest
- **Data strategy:** embedded (curated snapshot in `src/data/donations.ts`)

## Data Sources
- AEC Transparency Register — https://transparency.aec.gov.au/ (federal annual)
- NSW Electoral Commission — https://www.elections.nsw.gov.au/funding-and-disclosure/disclosures
- Victorian Electoral Commission — https://www.vec.vic.gov.au/candidates-and-parties/donations (real-time)
- Electoral Commission Queensland — https://disclosures.ecq.qld.gov.au/ (real-time)
- WA Electoral Commission — https://www.elections.wa.gov.au/elections/candidates-parties/disclosure
- Electoral Commission South Australia — https://www.ecsa.sa.gov.au/elections/funding-and-disclosure
- Tasmanian Electoral Commission — https://tec.tas.gov.au/
- Elections ACT — https://www.elections.act.gov.au/
- Northern Territory Electoral Commission — https://ntec.nt.gov.au/

## Architecture Decisions
- **Vanilla TypeScript over React**: A single-page tool with a search box, filter rail, and six view tabs doesn't need a component tree or virtual DOM. Re-rendering the whole content panel on every filter change is simpler, smaller (70 KB JS + 17 KB CSS total), and more predictable.
- **Embedded dataset over runtime fetch or pipeline**: No single API provides the unified cross-jurisdiction data this site needs, and building a pipeline to scrape 7 disparate regulator portals would be unreliable and brittle. An embedded curated snapshot (in `src/data/donations.ts`) ships a known-good dataset and gets clearly labelled in the About modal as illustrative, with links to every authoritative register.
- **Civic-tool light theme** (navy + teal + amber, clean whitespace) rather than dark/hacker aesthetic. The audience is journalists and citizens, not security analysts. Dark would feel conspiratorial and clash with the official registers the site mirrors.
- **No map view**: Donations are organised by jurisdiction (discrete categorical), not geography. A Leaflet map would be decorative and not informative. Jurisdiction comparison is presented as a card grid with thresholds, cadences, and lag times — far more useful.
- **localStorage persistence** for filter state, active tab, and sort direction. Restores user's working view on refresh. No cookies, no analytics, no third-party services.

## Test Results
- Tests written: 70 (across format, filter, aggregate, data integrity)
- Tests passed: 70
- Tests failed: 0

Tests cover:
- Number/currency/percent/FY formatting (including edge cases: NaN, Infinity, zero, negatives)
- Financial-year parsing and comparison
- Search query tokenisation (multi-token AND matching, case-insensitive, note-field search)
- Filter faceting (jurisdiction, party, donor type, amount range, fy range)
- Sort by all fields, both directions, immutability of input
- Aggregation functions (top donors, top recipients, totals by party/year, grand total, unique donor count)
- Dataset integrity (every record has valid jurisdiction code, valid party code, well-formed FY string, unique id)

## Build Status
- npm install: pass (113 packages)
- npm test: pass (70/70)
- npm run build: pass (1.04 KB HTML + 16.86 KB CSS + 70.05 KB JS, built in 97 ms)
- Local preview: pass (HTTP 200 on http://localhost:5199/)

## UI Verification (via Chrome MCP DOM inspection)
- Page title, hero headline, tagline rendering correctly
- Result summary: "Showing 276 of 276 records · $248.89M total disclosed · 180 unique donors"
- All 6 tabs render correctly:
  - **Donations table**: 276 rows, Mineralogy Pty Ltd $89.2M at top (sorted by amount desc)
  - **Top Donors**: Mineralogy Pty Ltd $179.16M (aggregated across years)
  - **Top Recipients**: United Australia Party $195.01M total received
  - **By Party**: UAP top with $195.01M
  - **By Year**: Correct chronological ordering from FY13-14 to FY23-24
  - **Jurisdictions**: 9 jurisdiction cards rendered (CTH, NSW, VIC, QLD, WA, SA, TAS, ACT, NT)
- Full-text search: "pratt" → 12 rows, 3 unique donors, $6.61M total
- Filter checkbox: VIC jurisdiction → 12 rows
- Glossary modal: opens with 11 entries
- About modal: opens with heading "About Donations AU"
- No JavaScript console errors

Note: Headless computer-use screenshot via Claude Desktop timed out because the permission dialog requires user approval and this is running as a scheduled task. DOM-level verification via Chrome MCP (which is already connected and authorised) is thorough and confirms the site renders correctly.

## Deployment
- Repo created: yes (ben-gy/au-donations, private)
- Initial commit: f6455f9
- GitHub Pages enabled: yes (build_type=workflow)
- DNS CNAME: au-donations → ben-gy.github.io (Cloudflare `benrichardson.dev` zone, grey cloud)
- GitHub Pages CNAME: au-donations.benrichardson.dev
- CNAME cycle performed: yes (triggers TLS cert issuance)
- Workflow run: 24474944535 (Deploy to GitHub Pages) — success
- Review branch pushed, PR created: ben-gy/au-donations#1
- Live URL verified: `curl -sI https://au-donations.benrichardson.dev` → HTTP/2 200, title tag matches

## Errors & Resolutions
- Computer-use `request_access` timed out after 300s (scheduled task has no user present to approve dialog). Resolution: skipped screenshot and relied on Chrome MCP DOM inspection (no approval dialog required).
- `https_enforced` flag on GitHub Pages still reports `false` after 5 minutes of polling, but `curl -sI https://au-donations.benrichardson.dev` returns HTTP/2 200 with a valid cert served by Fastly/GitHub. The flag appears to lag behind actual cert issuance. Live verification via curl is authoritative.
- No other errors encountered.

## Notes for future builds
- Chrome MCP DOM inspection is a reliable alternative to computer-use screenshots for unattended scheduled runs; no permission dialog and faster.
- The `https_enforced` Pages API field is unreliable for determining cert readiness — use `curl -sI https://<domain>` instead.
