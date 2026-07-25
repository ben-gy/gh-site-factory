# Build Log: Federal Election Results (au-election)
**Date:** 2026-07-25
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty. Scanned the registry (54 sites) and `gh repo list ben-gy` — the catalog is a mature set of Australian government/open-data explorers, but there was **no federal election results explorer**, a glaring gap: a crown-jewel AEC dataset that is highly searchable ("who won my seat"), geographic, and perfect for election-specific visual forms (Mackerras pendulum, preference-flow Sankey). Confirmed via WebSearch + WebFetch that the AEC Tally Room publishes the 2025 result (event 31496) as ~40 structured CSV downloads, all free and browser-fetchable with a browser UA. Confirmed the fleet already ships an authoritative Federal Electoral Divisions GeoJSON (150 real polygons, reused from au-mp-expenses), satisfying the never-hand-author-geometry rule.

## Site Details
- **Name:** Federal Election Results (index `country: AU`)
- **Repo:** ben-gy/au-election
- **Category:** government-transparency (index: data-explorers)
- **Audience:** voters looking up their own seat; journalists/students needing the numbers behind the headline; political followers wanting swings, margins and preference flows in one place
- **Stack:** Vanilla TypeScript + Vite 6 + Vitest; Leaflet for the map; everything else hand-rolled SVG
- **Data strategy:** pipeline — fetch AEC CSVs → parse → one reconciled `election.json` + boundaries in `public/data/`. Yearly cron (a federal election is ~3-yearly and final results never change; yearly is the slowest proportional cadence). Embeds the AEC `Generated` timestamp, not a run time, so re-runs are idempotent.

## Data Sources
- AEC Tally Room 2025, event 31496 — House CSV downloads (`results.aec.gov.au/31496/Website/Downloads/`): HouseFirstPrefsByCandidateByVoteType, HouseTppByDivision, HouseTcpByCandidateByVoteType, HouseMembersElected, HouseTppFlowByParty, HouseFirstPrefsByParty, HouseFirstPrefsByStateByParty, HouseTurnoutByDivision, HouseInformalByDivision, HouseSeatsWhichChangedHands.
- AEC / Digital Atlas of Australia — Federal Electoral Divisions boundaries (reused fleet `electorates.geojson`, 150 real polygons, 1:1 name join verified).

## Architecture Decisions
- **Vanilla TS** (not React): a single-page multi-view dashboard with hand-rolled SVG charts — smaller bundle, simpler.
- **Winner/margin from TCP, notional line from TPP.** The two-candidate-preferred file gives the actual final two per seat (so teal/Greens "non-classic" wins are correct); the two-party-preferred file gives the notional Labor-v-Coalition line used by the pendulum and swing map. Both are presented with the distinction made explicit.
- **Swing normalisation.** AEC's TppByDivision `Swing` is signed toward the leading party (`PartyAb`); normalised to swing-to-Labor as `PartyAb==='ALP' ? Swing : -Swing`.

## Data traps handled
1. **AEC banner line.** Every download's line 1 is a metadata banner, not the header — `parseAecCsv` strips it.
2. **Degenerate swings (the big one).** Three seats non-classic in 2022 (Brisbane, Bendigo, Nicholls) have no comparable prior two-party figure, so the AEC reports `Swing` equal to the current ALP TPP percentage as a sentinel. Taken at face value these become absurd +59 / +51 / −36 "swings" that skewed the national average, the swing rankings, the swing histogram and the map. Detected (`|swing − ALP%| < 0.05`), nulled, excluded from all swing views, drawn grey on the map with a legend entry, and the national swing taken as the correct aggregate (55.22 − 52.13 [2022 national result] = **+3.09**) rather than the mean of divisional swings.
3. **Informal tally rows** in the per-candidate first-prefs file (Surname `Informal`) filtered out of candidate lists.
4. **Quoted embedded commas** (e.g. "Shooters, Fishers and Farmers Party") — RFC-4180 reader.

## Reconciliation gate (fails the build on drift)
150 divisions; every seat has a member, a 2-row TCP pair and a positive TPP; the TCP winner group == the elected member's group in every seat; seats-by-group sum to 150; first-preference party `Elected` column sums to 150; national TPP ALP within 0.3 of ~55.2%; national swing within 2.5–3.7.

## Views (9)
Overview (parliament horseshoe, crossbench between the majors) · Seat map (Leaflet, winner/TPP/swing) · Mackerras pendulum + uniform-swing simulator · Preference-flow Sankey · First preferences + party×state matrix · Rankings (6 metrics) · Explorer (searchable/sortable) · Distribution (margin + swing histograms, click-through) · Insights (auto-detected). Hash-linkable per-seat drawer.

## Test Results
- Tests written: 56 (parser + AEC banner + num/partyGroup; positional layout tests for binValues/stackBands/niceMax; election.json integrity + pendulum + seatsFlippedBy + insights + the swing-null trap)
- Tests passed: 56 / failed: 0

## Build Status
- npm install: pass
- npm test: pass (56)
- npm run build: pass (tsc clean; bundle 211 KB / 64 KB gzip, mostly Leaflet)
- Local preview: pass

## Deployment
- Repo created: yes (ben-gy/au-election)
- GitHub Pages enabled: yes (workflow build)
- Cloudflare DNS CNAME: created (au-election → ben-gy.github.io)
- Deploy workflow: green (npm ci + test + build + deploy)
- Custom domain live over HTTPS: yes (`https_enforced=true` after one CNAME cycle)
- Production verified: live bundle byte-identical to local dist; 150 map polygons; per-seat drawer above Leaflet (Dickson → Ali France); zero console errors. Full real-click + all-view + 375px-overflow verification done on the byte-identical local production build.
- PR: created for review (adds REVIEW.md).

## Errors & Resolutions
- **Bar fills collapsed to 0px.** `.barrow .fill` was a `<span>` (default `display:inline`), so `width/height` didn't apply — every overview/first-prefs/rankings bar rendered as an empty track. Fixed with `display:block`. Caught by screenshot inspection (not by any test).
- **Absurd swings.** See trap 2 above — caught by looking at the Insights "biggest swing +59.0" and cross-checking the raw AEC CSV.
- **Mobile overflow on Rankings.** The 6-button metric selector (458px) overflowed 375px. Fixed by making `.seg` scroll locally (`overflow-x:auto; min-width:0`).
- **Persistent tooltip/drawer across navigation.** Added `hideTooltip()` and `closeSeatDrawer()` on view change.
