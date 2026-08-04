# Build Log: Work Injuries
**Date:** 2026-08-04
**Status:** deployed (with two reported limitations — see Errors & Resolutions)

## Idea Source

IDEAS.md was empty, so the idea was researched. Eight candidate domains were
investigated in parallel by agents required to verify every claim with real HTTP
requests — workplace safety, water markets, agriculture, scams, protected areas,
fisheries, aviation safety and ambient air quality — then scored by three
independent judges on buildability, novelty and audience value.

Ranked: **worksafety 25.5**, protectedareas 25, agriculture 24.5, airquality 21.5,
watermarket 21, fisheries 19, scams 15.5.

Workplace safety won on the cleanest sweep of the novelty tests: nothing in the
64-site fleet touches occupational injury, workers' compensation or WHS regulator
enforcement (au-mortality is AIHW all-cause death, au-crashes is road fatalities,
au-jobs is unemployment), and Safe Work Australia is a new publisher for the
factory. All five source URLs verified HTTP 200 on plain curl, CC BY 4.0 with
commercial reuse explicitly permitted by the publisher's own copyright page.

## Site Details
- **Name:** Work Injuries
- **Repo:** ben-gy/au-worksafe
- **Category:** public-safety
- **Audience:** the ~100,000 elected Health and Safety Representatives who have a statutory role and nowhere to look up their own industry's risk profile; WHS officers and return-to-work coordinators; injured workers benchmarking a claim; small-business owners wondering why their premium is what it is; union organisers; journalists.
- **Stack:** vanilla TypeScript + Vite 6, Leaflet 1.9 for one supporting map
- **Data strategy:** pipeline (build-time), **manual-only** — see limitation 2

## Data Sources

| Source | Licence |
|---|---|
| Traumatic Injury Fatalities detailed data file | CC BY 4.0 |
| National Dataset for Compensation Based Statistics detailed data file | CC BY 4.0 |
| Jurisdictional Comparison detailed data file | CC BY 4.0 |
| Beta Occupational Hazards Dataset (Dec 2023) | CC BY 4.0 |
| TIF time series, 2021 vintage — the only pre-2015 history | CC BY 4.0 |
| ABS ASGS state boundaries (vendored from `patterns/geo/`) | CC BY 4.0 |

## Architecture Decisions

**Vanilla TS.** Nine views, one shared render/mount contract, no routing library
needed. Only Leaflet is added, for a map that is deliberately a supporting view.

**A dependency-free xlsx reader.** Everything Safe Work Australia publishes is
`.xlsx`. `pipeline/xlsx.mjs` unzips the archive from the central directory and
walks the sheet XML, so the data workflow needs no `npm ci` and cannot drift.

**Three views were deliberately NOT built**, and the site says why in place:
- **No TOOCS causal-chain Sankey.** The four codings — what went wrong, how the
  body was hurt, which part, what object — are published as four independent
  tables of totals with no record linking them. A flow diagram from marginals
  shows connections nobody measured.
- **No hero choropleth.** The finest geography in the entire corpus is
  state/territory, and a "jurisdiction" is a compensation scheme, not a place.
- **No case-level prosecutions database.** The register has no bulk file.

**The signature view crosses two measures the publisher never crosses**: how
often an industry injures people against what each injury costs, on a log cost
axis with data-defined median crosshairs. It separates frequent-but-recoverable
industries from rare-but-ruinous ones, which any claim-count-driven safety
programme misses entirely.

## Traps handled (each found in the real files, not assumed)

1. **Self-closing cells stealing the next cell's value.** `<c\b[^>]*>` matched
   before the self-closing form, so `[^>]*` swallowed the trailing slash of an
   empty `<c r="A1"/>` and the parser consumed the NEXT cell as its body. **This
   shipped**, fabricating a frequency rate of 33.00 for "99 Inadequately
   described" where the source publishes NP, and simultaneously collapsing the
   hazards dataset's 13 published families to 2. Found by the review, fixed by
   ordering the self-closing alternative first with a lazy `[^>]*?`.
2. **Four code shapes that read as top-level parents.** `8_ Mental stress not
   further defined` alone added 1,718 claims to a 106,496 total; also `34/35`
   merged codes and `&`/`@` sentinel rows.
3. **`NP` is not zero** — it means "fewer than five", and the entire latest-year
   column of every median table is NP because compensation accrues after the year.
4. **The hierarchy gate was vacuous.** "Children must sum to their parent" skipped
   any group with a suppressed child — 189 of 190 groups — leaving ONE real
   check. Rewritten as a suppression-aware bound (`sum(known) ≤ parent ≤
   sum(known) + 4·nSuppressed`, from the publisher's own under-five rule): 190
   real assertions.
5. **The cross-vintage revision.** The two releases disagree on 2016/2018/2019/
   2020/2021 and AGREE on 2015 and 2017 — so a stitch rule validated on either of
   those passes while being completely wrong. A test demonstrates exactly that.
6. **The premium industry-header row is a MAXIMUM**, not a national rate: 95/95
   cells equal the jurisdiction maximum, 0/95 equal the AUST row. Dropped, with a
   gate re-asserting the property every build.
7. **The cover page lies** — the fatalities workbook advertises two gig-worker
   tables that do not exist. Sheets enumerated from `xl/workbook.xml`.
8. **The hazards sheet is not all hazards** — its last four columns are employment
   and real claim rates, not 0–100 scores. A test caught it; used correctly they
   removed the rollup and its "no employment weights" caveat entirely.

## Test Results
- Tests written: **160**, all passing
- 34 parser (incl. four self-closing-tag regressions), 20 gate, 44 positional layout, 14 overlay-contract, 48 view-integration against the real committed data
- **Every gate is proven to FAIL on the fault it claims to catch**, with injected faults placed where the failure actually lives

## Build Status
- npm install: pass
- npm test: pass (160/160)
- npm run build: pass
- Local preview: pass
- Live production verification: pass

## Deployment
- Repo created: yes — https://github.com/ben-gy/au-worksafe
- GitHub Pages enabled: yes
- Deploy workflow: **GREEN** on every push
- Custom domain: **NO** — see limitation 1
- Directory entry live on main: yes
- Live URL: https://ben-gy.github.io/au-worksafe/

## Pre-deploy review

A 31-agent adversarial review across four lenses (data accuracy, misleading
presentation, code correctness, a11y + mobile). Every finding was handed to its
own agent instructed to **refute** it. **27 raised → 3 refuted, 24 confirmed and
all 24 fixed.**

The one that mattered most was #1 above: real data fabrication that had already
shipped. Others: a card titled "Where mental-stress claims land" ranking all
serious claims (no per-industry psychological count exists in the release); four
raw-volume rows shaded on the same worse-is-darker ramp as five size-adjusted
rates, reading as an enforcement league table when it mostly measured jurisdiction
size; a dumbbell described as dollars-against-weeks when both dots were weeks; a
"most expensive body part" list topped by a 15-claim median; an age table
asserting a per-capita fact with no denominator; the ACT premium showing as a
dash because that table publishes it under "ACTPrivate"; a leaked Leaflet map and
ResizeObserver per render; Victoria/Queensland/Tasmania labelled "V"/"Q"/"T"; and
an a11y batch — all three overlay ✕ controls bound to `pointerdown` alone and so
dead to keyboards, no focus trap on a modal declaring `aria-modal`, mouse-only
drill-down rows, a map drill-down unreachable because Leaflet never gives a vector
path a tab stop, `role="img"` hiding 312 and 64 focusable dots from the
accessibility tree, treemap values at 1.32:1, and matrix numerals darkened into
their own fill by `mix-blend-mode`.

## Errors & Resolutions

**1. No custom domain — Cloudflare DNS record cap.** `au-worksafe.benrichardson.dev`
could not be created: the `benrichardson.dev` zone holds exactly **200 records**,
the free plan's hard limit, and the API returned `Record quota exceeded` (81045).
Rather than ship a site that resolves nowhere, it serves from
`https://ben-gy.github.io/au-worksafe/` — vite `base` set to `/au-worksafe/`,
`public/CNAME` removed, and every absolute URL in the head, sitemap, robots.txt
and README updated to match.

**19 of the zone's CNAMEs point at `ben-gy.github.io` with no matching repo** and
look reclaimable: `artemistracker`, `au-build-approvals`, `au-cpi-explorer`,
`au-insolvency-tracker`, `au-name-trends`, `au-ndis-register`, `audams`,
`bgwipe-cutout`, `castwell-cast`, `emberwake-run`, `facet-dice`, `metascrub-app`,
`skein-untangle`, `tiltwell-level`, `veilpix-stego`, `vic-rent-atlas`, `voynich`,
`wavewell-audiogram` (`www` is also unmatched but is presumably intentional).
Deleting the user's DNS records was not mine to do, so this is reported rather
than acted on. **This will block every future site too** until records are freed
or the zone is upgraded.

Moving this site across once a record is free is three steps, documented in the
README: add the CNAME, restore `public/CNAME`, set `base` back to `'/'`.

**2. The data workflow cannot run on CI.** `safeworkaustralia.gov.au` refuses
GitHub-hosted runners outright. With the resolver pinned to `ipv4first`, browser
headers set, a 120-second timeout and four backed-off retries, **zero of the five
downloads completed** — every one failed as a bare `fetch failed` — while the
identical code fetches all five in about twenty seconds locally. The host appears
to refuse datacentre address space.

A yearly cron would do nothing but email a failure once a year, so the schedule
was removed and `workflow_dispatch` kept for a future retry. The annual refresh is
`npm run collect && git add public/data && git commit && git push`. The committed
data is what the site serves, so nothing is broken in the meantime.

**3. Fixed during the build.** The archive workbook is transposed relative to the
current one (years are rows, and the year drifts between column 0 and column 1),
so it is scanned directly rather than through `parseTable`.
