# Build Log: Product Recalls
**Date:** 2026-08-07 (deployed 2026-08-08 UTC)
**Status:** deployed

## Idea Source

Researched. `IDEAS.md` was empty, so an 8-agent scout workflow investigated eight
candidate domains against real fetch evidence — ACCC product recalls, AFCA firm
complaints, ACARA schools, ATSB occurrences, the CASA aircraft register, ABS
Criminal Courts, ABARES agriculture, and a client-side national air-quality feed.

A judge agent picked ACCC product recalls. Two candidates were killed by
**licence, not data**, and are recorded here so they are not re-scouted: AFCA's
per-firm complaints datacube (terms forbid redistribution without written
approval) and ACARA My School (clause 6.4 explicitly excludes distribution "on a
publicly accessible website"). In both cases the fetch *succeeded* — that is
precisely the trap.

## Site Details
- **Name:** Product Recalls
- **Repo:** ben-gy/au-recalls
- **Category:** public-safety
- **Audience:** primarily a person holding a product who wants to know if it is
  recalled; secondarily journalists and compliance people who want the shape of
  the register
- **Stack:** vanilla TypeScript + Vite, no runtime dependencies (84 KB JS, 29 KB gzipped)
- **Data strategy:** pipeline, **monthly** cron (9th, 04:23 UTC). Both registers
  publish continuously; monthly is the fastest cadence the factory allows, and
  the crawl is incremental against a committed parsed store.

## Data Sources
- ACCC Product Safety recall register — `productsafety.gov.au`, Drupal Views
  listing + per-recall detail pages. 8,218 recalls. CC BY 4.0.
- ACCC product-category facets — the 119-term taxonomy **and the register's own
  count for each term**, used as an external reconciliation anchor.
- Vehicle recall register — `vehiclerecalls.gov.au`, Department of
  Infrastructure. 5,529 campaigns, 21,957,382 affected units. Conditional CC BY
  4.0, attributed separately; VIN lists deliberately not reproduced.

## The finding

Button-battery hazards went from **1.9% of recalls in 2018 to 19.3% in 2025**
(peaking at 28.2% in 2023) while the **annual total fell from 474 to 254**.
Australia is not recalling more things; it is recalling a different set of
things. Neither register publishes any aggregate of its own.

## Architecture Decisions

- **Crawl, not download.** Neither register publishes a bulk file, an API, or a
  data.gov.au extract.
- **The raw HTML is never committed.** The crawl is ~1 GB; the repository carries
  `pipeline/parsed/` (16 MB of extracted fields), which is also the incremental
  key that keeps a monthly CI refresh cheap.
- **`public/data` is derived at deploy time**, not committed. It is a pure
  function of the parsed store, so committing it would duplicate 18 MB — and
  deriving it in CI means **every deploy re-runs the reconciliation gates**.
- **No map**, deliberately. The "where sold" field mixes a national flag, a sales
  channel and occasionally a state in one list, and most recalls are national; a
  choropleth would mostly redraw where Australians live.
- **The two registers are never summed.** Different regulators, categories and
  date meanings.

## Three discoveries, each found by a gate failing

1. **The ACCC's list is not homogeneous.** 1,382 entries are FOOD recalls whose
   URLs redirect to FSANZ and publish no ACCC hazard detail at all; 99 are
   VEHICLE recalls that redirect to the vehicle register and carry a REC number
   (all 99 verified to resolve there). Reading all three kinds as "an ACCC recall
   with empty fields" produced 1,383 phantom unclassifiable rows and put FSANZ's
   site navigation into a product-safety search index.
2. **Detail pages live under three URL shapes** —
   `/search-consumer-product-recalls/`, `/recalls/` and `/node/NNNNN`. A parser
   accepting only the common one lost one recall in 2006 and one in 2007.
   Invisible in a global total; obvious the moment the crawl is checked
   year-by-year against the register's own per-year counts.
3. **`layout=full_width` is load-bearing.** Without it, page 0 returns the view
   shell with an *empty* results region while still reporting "1 - 24 of 8,218
   items shown". A crawler that shrugged at an empty first page would publish the
   register missing its **25 most recent** recalls.

Also handled: `page` is honoured only in the query string (any other way returns
a valid-looking page 0 every time); the default relevance sort is not stable
across pages, so the listing is crawled a **year at a time** with a month-level
fallback; and a `field--label-hidden` field bled the next heading into its value,
appending "Identifying product features" to 2,880 traders values.

## Gates

Fifteen, all passing. Two are anchored on numbers this pipeline did not
compute — the register's own per-year totals and the ACCC's own per-category
facet counts — and one on the cross-register REC join. The year and category
gates account for *named* unreachable records exactly rather than tolerating a
fudge factor: two recalls are dead links in **both** registers (`/node/19706` →
REC-003902 and `/node/20135` → REC-001401, both 404).

Two gates were rewritten after review because they had no teeth:
- "detail coverage" was `parsed + missing === total`, which passes happily on a
  dataset that is a quarter empty. It now has a 99.5% floor, and the vehicle side
  gained the same (its identity gate was satisfied by an *empty* register).
- "registers stay separate" asserted `true` with a sentence — and asserted the
  wrong thing. It is now a real check that every cross-listed REC number resolves.

## Test Results
- Tests written: **228** (parse 75, gates 39, layout 33, overlay 19, views 62)
- Tests passed: 228
- Tests failed: 0

The view tests render **every** view against the real committed data and assert
no NaN in any SVG coordinate, an accessible name on every chart, no `role="img"`
on a chart containing focusable marks, no native `<title>` used as a tooltip, and
no `role="button"` outside the tab order.

## Build Status
- npm install: pass
- npm test: pass (228)
- npm run build: pass
- Local preview: pass
- **Live production verification: pass**

## Adversarial pre-deploy review

53 agents across four lenses (data accuracy / misleading presentation / code
correctness / a11y + mobile), every finding independently verified by an agent
told to **refute** it. **49 raised, 8 refuted, 41 confirmed and fixed.**

The two criticals that would otherwise have shipped:

1. **The search index omitted the hazard and defect prose entirely.** 1,277
   recalls whose notice says "choking" and 576 that say "electric shock" returned
   *nothing* — while "Choking / small parts" and "Electric shock" sat in the
   hazard dropdown beside the box — and the empty state told the reader in bold
   that *the ACCC* had published no such recall. On a site whose one job is
   telling a worried parent whether their product is listed, that is the worst
   failure available. Both the index and the copy are fixed; the empty state now
   describes what was searched and never makes a claim about the register.
2. **The supplier network published an edge-weight sum labelled "Appears on N
   recalls"** — up to 22× the true count — on the same size scale as the retailer
   nodes it existed to be compared against. Both sides are now distinct recalls.

Other confirmed fixes worth naming: the hover tooltip was never hidden (the
pattern toggles `.visible` and nothing styled it, so the first tooltip followed
the pointer forever); `attachSvgZoom` dereferenced `svg.viewBox.baseVal`
unguarded, which jsdom does not implement, so the entire view test suite crashed
the moment real data existed; the retailer network presented suburbs, states and
"See traders list below" as retailers and split Big W across four nodes; the
skip link's `href` was a route, so "skip to content" threw the reader back to
Search; `touch-action: none` created a ~600px band a phone could not scroll past;
`closeAllOverlays` bypassed `close()` and leaked a capture-phase keydown listener
per navigation; every taxonomy, matrix, supplier and histogram drill-down
navigated to a *free-text* search that found a different number than the block
just clicked (all now filter on ids and reproduce the figure exactly — verified:
category 1,714 = 1,714, supplier 93 = 93, shelf bin 789 = 789); and a batch of
contrast failures (all 13 hazard pill colours now pass white-on-fill AA at ≥
4.74:1; icicle sub-category labels were white on a 60%-opacity fill at 2.34:1 and
are now ink on a lightened fill; the matrix text threshold sat at luminance 0.42,
handing white text to cells where it measured 2.25:1).

Also corrected as a matter of honesty rather than a bug: the hazard class is
inferred by this site, **except** button batteries, which come from the ACCC's own
retro-applied tag — the chart said "inferred from the wording of each notice"
without that exception, and the drawer showed the class as a coloured pill beside
official register fields with nothing marking it as derived.

## Live verification (production URL)

All nine views render at `https://au-recalls.benrichardson.dev` with zero console
errors, zero NaN coordinates and zero page-level horizontal overflow at both
1440px and a true 375px viewport. Live bundle `index-Dy70Y-qh.js` matches the
local dist. `/`, `data/meta.json`, `data/recalls.json`, `data/stats.json`,
`data/vehicles.json`, `data/detail/0.json`, `og.png`, `robots.txt`,
`sitemap.xml`, `favicon.svg`, `third-party-notices.txt` and the IndexNow key all
return 200.

Real trusted clicks on production: typing "button battery" into the search box
filtered 8,216 → 266 and updated the URL; a real click opened the dossier for
"Secret diary pen with UV light" with the defect, the hazard, the emergency
numbers and "What to do". **The full dismissal contract passes on the live site**,
asserted by computed style and re-opened between each exit: ✕ (44×44), scrim by
mouse `pointerdown`, scrim by a real `pointerType:'touch'` sequence with no
double-fire reopen, and Escape — plus focus moved into the panel, restored on
close, the `?r=` deep-link param cleared, and the body scroll lock released. The
About modal opened **from the network view** paints fully above it and closes on
Escape. Selecting a network node dims 537 unrelated elements.

## Errors & Resolutions

- **Rate-limited by the ACCC at ~6,400 requests** (HTTP 403 on everything).
  Stopped crawling rather than hammering a blocked host, added 5s/15s/45s/135s
  backoff on 403/429, dropped concurrency 8 → 4, and made a page that cannot be
  fetched a *counted* miss rather than a crash. Added `--vehicles-only` /
  `--accc-only` so the unblocked register could be crawled meanwhile. The block
  lifted after ~25 minutes and the crawl completed.
- **ENAMETOOLONG** two thousand pages in: one product slug runs to 240 characters
  and exceeded the 255-byte filename limit. Long keys now get a truncated prefix
  plus a hash of the whole key.
- A `<textarea>`-wrapped AJAX response (missing `X-Requested-With`), and an
  empty-year slice that legitimately returns no counter at all — distinguished
  from a broken endpoint by an explicit `view-empty` marker, because conflating
  them would read a broken crawl as an empty register.

## Deployment
- Repo created: yes — https://github.com/ben-gy/au-recalls
- GitHub Pages enabled: yes
- DNS (Cloudflare CNAME): yes — 194/200 records in the zone, 6 slots remain
- TLS certificate: approved on the first poll
- Deploy workflow: **green**
- Directory entry live on main: yes

## Honest limitations

- **13.7% of recalls fall in the "Other" hazard class.** Roughly half of those
  have no hazard or defect text at all in the register. The share is stated on
  the chart it affects.
- **The network view still has 33 sub-12px focusable nodes at a 375px viewport.**
  The graph scrolls locally at a 760px minimum (down from 225 such targets), and
  only hub nodes are focusable, but small nodes remain small on a phone.
- **Retailer name variants remain** where merging would be a guess — "Bunnings"
  and "Bunnings Warehouse Stores" stay separate, consistent with the deliberately
  conservative merge rule that refuses prefix matches.
- **Two recalls are unreachable** in both registers and are named, counted and
  excluded rather than silently dropped.
- **`vehiclerecalls.gov.au` is unreachable from a GitHub runner.** This is the
  au-worksafe pattern again. The first CI pipeline run failed after eighteen
  minutes with every request to that host — including page 0 of the listing —
  aborting on the request timeout, while the same URLs load fine from a laptop.

  Two things were done about it. First, the diagnosis mattered: the ACCC half of
  that run had **completely succeeded**, passing its listing gate (8,218 ==
  8,218) and demonstrating that the committed parsed store makes a refresh cheap
  exactly as designed — `detail pages: 8216 cached, 2 to fetch`. Losing that to a
  failure on the *other* register was the actual bug. The vehicle crawl now fails
  **soft**: it reports loudly and the run continues with the committed vehicle
  store untouched. Verified against a simulated outage — the store survives
  intact at 5,529 campaigns and all 15 gates stay green. A crawl returning
  *fewer* rows than we hold is still caught by the vehicle coverage gate, so this
  trades a hard failure for a stale half, never a silent one.

  Second, the vehicle listing was made incremental anyway (it re-walked all ~280
  pages every run; it now stops after three pages' worth of consecutive
  already-known campaigns — verified at 3 pages instead of 280). That does not
  help on CI, where page 0 itself is unreachable, but it makes the local refresh
  fast.

  **Net effect:** the monthly cron refreshes the ACCC register, which is the half
  that actually changes weekly. The vehicle register must be refreshed locally
  with `node pipeline/collect.mjs --vehicles-only`, which takes about a minute.
  Nothing is broken in the meantime: the committed store is what the site serves,
  and the deploy workflow regenerates and re-gates it on every push.
