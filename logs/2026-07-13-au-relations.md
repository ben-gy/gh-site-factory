# Build Log: Foreign Relations
**Date:** 2026-07-13
**Status:** deployed

## Idea Source
IDEAS.md (second entry): *"Intergovernmental Relations (AU) — machinery of government from Australia's perspective but outward-facing: how Australia formally relates to other governments. Treaties (Australian Treaties Database / austlii treaty library), diplomatic posts and DFAT embassy network, free trade agreements, international organisation memberships, sister-agency arrangements, and bilateral relationship views per country. Network graph of Australia's government-to-government relationships."*

The **first** IDEAS.md entry (Machinery of Government AU) was logged to EXPANSION_IDEAS.md instead of being built — it substantially overlaps the deployed **au-gov** (AGOR federal explorer); the new angles (state structures, AAO/MoG change history, ministers) are enhancements to that site.

## Site Details
- **Name:** Foreign Relations
- **Repo:** ben-gy/au-relations
- **Category:** government-transparency
- **Audience:** Journalists, IR students/researchers, policy staffers, and the public checking Australia's formal ties with a specific country
- **Stack:** Vanilla TypeScript + Vite + Leaflet + Vitest
- **Data strategy:** pipeline — quarterly cron (`23 6 9 1,4,7,10 *`); the ATD publishes treaty actions irregularly (~monthly at fastest) and the missions/FTA pages change a few times a year, so quarterly is proportional

## Data Sources
- DFAT Australian Treaties Database — **discovered an undocumented JSON search API** at `https://docs.dfat.gov.au/api/search` (POST `{keyword, page, facets, dateFilters}`); the old info.dfat.gov.au Lotus-Notes system now redirects to this Vue SPA. 4,522 records total; 4,512 collected (10 stragglers lack usable dates)
- DFAT embassies & consulates overseas page (parsed: resident posts, consulates, "see X" accreditation referrals, Canadian shared-consular flags)
- DFAT trade agreement pages — curated `pipeline/data/ftas.json` (19 in force + 2 negotiating)
- Curated organisation memberships (`organisations.json`): G20, OECD, APEC, Commonwealth, PIF, EAS, Quad, AUKUS, Five Eyes, IPEF + universal bodies
- Country metadata table (`country-meta.json`): ~240 entities incl. aliases, ISO codes, regions, centroids, historical states (USSR, Czechoslovakia, Yugoslavia, GDR, Rhodesia & Nyasaland, Aden), territories
- Natural Earth world GeoJSON (johan/world.geo.json, ISO3 ids) for the Leaflet choropleth

## Architecture Decisions
- **Vanilla TS** — seven views but a single-page hash-routed app; no component-tree complexity that would justify React
- **Pipeline over runtime-fetch** — the ATD API is POST-only (unknown CORS posture), pagination is unstable, and 227 requests per load would be hostile; a quarterly Actions pipeline commits static JSON instead
- **Year-sliced collection with Id de-dupe** — the ATD API's pagination ordering is unstable between requests; slicing by `DoneAtDate` year keeps each slice small and re-looping pages until `distinct == totalCount` makes collection deterministic; an unfiltered sampling pass catches undated records
- **Per-country multilateral counts dropped** — the ATD stores empty `Countries` arrays for all 2,470 multilateral treaties, so "shared multilateral" per country is impossible; the UI states this explicitly and all per-country counts are bilateral
- Treaties corpus (2.1 MB JSON) loaded lazily; countries/stats (~200 KB) up front

## Test Results
- Tests written: 56 (utils, analysis/insights, pipeline categoriser/status/trimmer/post-parser/missions-parser)
- Tests passed: 56 · failed: 0

## Build Status
- npm install: pass · npm test: pass · npm run build: pass (tsc + vite, 62 KB gzip JS)
- Local preview: pass — verified all 7 views + drill-down + mobile via browser preview

## Deployment
- Repo created: yes (ben-gy/au-relations)
- GitHub Pages enabled: yes (workflow build; deploy run green)
- Custom domain: au-relations.benrichardson.dev — Cloudflare CNAME created via API, cert cycle run, **https_enforced=true confirmed**
- PR created: https://github.com/ben-gy/au-relations/pull/3 (assigned ben-gy)

## Errors & Resolutions
1. **DFAT missions fetch timed out** with a custom pipeline User-Agent (WAF tarpit). Fixed with browser-like headers + retries + `--missions`/`--treaties` CLI flags for partial re-runs.
2. **ATD pagination unstable** (same query, different order each request; page overlap). Fixed with year-sliced queries + Id de-dupe + re-loop until complete.
3. **~100 unmatched country names** on first aggregation — resolved with alias table additions (historical states, territories, org-name variants, "China (People's Republic of)" style), split rules for multi-party strings, and an ignore-list for junk missions rows.
4. **Missions parser grabbed the wrong parenthesis** for countries whose names contain parens ("China (People's Republic of)") and broke on DFAT's Singapore typo (missing closing paren). Fixed by parsing only after the `</strong>` name and requiring post-like keywords; regression tests added.
5. **Multilateral treaties have no member-country data** in the ATD (see Architecture Decisions) — UI reworked mid-build to drop the misleading per-country metric.
