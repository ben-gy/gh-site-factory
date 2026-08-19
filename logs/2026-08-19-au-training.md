# Build Log: Training Register
**Date:** 2026-08-19
**Status:** deployed

## Idea Source

Researched. `IDEAS.md` was empty, so six candidate domains were investigated in parallel — vocational
education, aviation safety, schools, ambient air quality, industrial relations, and maritime/fisheries
— each by an agent required to verify every source with a real fetch and to report honestly what it
could not verify. Three independent judges then scored the survivors on different lenses (product
utility, evidence quality, visualisation craft).

**Winner: the National Training Register (2 of 3 judges).** The evidence and craft judges both picked
it; the product judge picked air quality but ranked the register second.

Rejections worth recording, all found by fetching rather than recalling:

- **Schools (ACARA)** — rejected outright (scored 15/100). The school-level data is behind a Data
  Access Program application form.
- **NCVER** (the apprentice/student volumes originally sought) — `www.ncver.edu.au` returns HTTP 403
  behind a Cloudflare JS challenge on *every* path including `/robots.txt`, and VOCSTATS is a login
  wall. No CI runner will ever get past it. This is why the site covers the supply side only.
- **ATSB occurrence database** — `data.atsb.gov.au` is NXDOMAIN; the publisher states in writing
  "Can I download your whole database? No."
- **Air quality** — genuinely viable and the runner-up, but EPA Victoria returns HTTP 401
  (`WWW-Authenticate: AzureApiManagementKey`), WA has no open ambient series, and SA stops in 2020.
  A "national" air quality site without Melbourne is a permanent apology. Logged as a future idea.

## Site Details
- **Name:** Training Register
- **Repo:** ben-gy/au-training
- **Category:** education (VET regulation — a new domain for the fleet)
- **Audience:** prospective VET students and parents checking a college before paying; employers
  looking for a provider who can deliver a qualification in their state; RTO compliance staff;
  journalists and analysts on skills shortages and enforcement
- **Stack:** Vanilla TypeScript + Vite 6, Leaflet 1.9 (the only runtime dependency), hand-rolled SVG
- **Data strategy:** pipeline, **monthly cron** (day 9, 16:43 UTC). The register changes every working
  day, but the aggregate picture moves slowly; monthly is both the factory's fastest permitted cadence
  and genuinely proportional.

## Data Sources

The National Training Register's NTR Web API — public, unauthenticated (all eight Swagger specs have
an empty `securitySchemes` block), CC BY 4.0 per `training.gov.au/copyright-information`:

| Endpoint | What it gives | Volume |
|---|---|---|
| `/api/search/organisation` | Every organisation ever on the register | 13,146 |
| `/api/search/training` (Type/Id 2) | Every training-package qualification | 8,034 (1,164 current) |
| `/api/training/{code}/delivery` | The RTO↔qualification scope graph | 39,182 raw scope rows |
| `/api/organisation/{code}/regulatorydecision` | Sanctions, conditions, cancellations | 676 decisions |
| `/api/organisation/{code}/restrictions` | Restrictions in force | 449 |

Plus the fleet's ABS ASGS state boundaries (CC BY 4.0) for the map.

## Architecture Decisions

**Why not the bulk CSV export.** `POST /api/organisation/csv` returns a real 9.6 MB CSV anonymously
and is deterministic — but it is **incomplete** (12,593 distinct codes against a register of 13,146)
and it ships **fourteen columns of personal contact details**. The paged JSON API, harvested properly,
is both more complete and lets the parser never read those fields at all.

**Why the supply side only.** NCVER is unreachable, so there are no student, enrolment or completion
counts anywhere in this build. That limit is stated on the Method page rather than papered over.

## The two API traps that would have shipped a confidently wrong site

**1. Offset paging over `/api/search/*` is LOSSY under the default ordering.** The result order is
stable for a given offset but not across offsets, so a sweep returns the right *number* of rows and
the wrong rows. Measured on the organisation index: one sweep at `pageSize=500` gave 13,146 rows but
only **12,114 distinct**; at `pageSize=200`, 10,803 distinct; ~2,400 records appeared in one sweep but
not the other. **A `rows.length === totalCount` check passes throughout**, because the count is right
and only the identities are wrong. Nearly a thousand providers were missing — including the University
of Tasmania — from a harvest that looked complete. It announced itself only as 189 dangling delivery
edges, caught by a gate.

First fix was union-until-dry across varying page sizes, which converged to exactly 13,146. The
verification agent then found the better fix: **`orderBy` makes paging deterministic**, and a single
ordered sweep returns exactly `totalCount` with no duplicates and no skips. The pipeline now does that
and runs a second sweep at a deliberately awkward page size that must add nothing.

**2. `/delivery` truncates silently.** It has no `totalCount` at all, returns exactly `pageSize` rows
and a `links[].rel == "next-page"`. Reading one page reports BSB50420 — the country's most widely
offered qualification — as having **721** providers instead of 809. Exactly one of the 1,164 current
qualifications exceeds 1,000 rows, and it is unfortunately the headline one. Every request now follows
the link chain to exhaustion, and the build refuses any result whose row count is an exact multiple of
the page size with no terminating page.

## Adversarial verification

Three agents were tasked with **refuting** the headline figures: one re-derived all of them from
scratch with its own clean-room Node harvest, one attacked the definitions, one audited privacy.

**Confirmed to the exact digit:** 13,146 organisations; 8,034 qualifications; 1,164 current; 3,886
fully-current RTOs; **144 orphan qualifications**; 137 with exactly one; median 5; **809 for BSB50420**;
and every per-state figure (NSW 198, VIC 200, QLD 208, SA 225, WA 199, TAS 223, NT 228, ACT 228).

**Two real defects found and fixed:**

1. **"440 have five or fewer" was wrong.** 440 is the one-to-five bucket; five-or-fewer is 585,
   because the 144 zeros belong in it. As drafted the site would have contradicted its own headline
   on the same screen. Both populations are now named explicitly and separately in the data and the UI.
2. **`liveEdges` was mislabelled.** 29,331 is all live scope entries *including* 183 assess-only —
   the very thing the site's own rule excludes. The published figure is now 29,056 teaching edges with
   assess-only reported separately, computed from the same predicate as every provider count so the
   numbers on different views add up.

**One stale-flag trap disclosed:** `isHistoric` is unreliable — decisions marked not-historic and
"In effect" with end dates years past (Gordon Institute of TAFE and Box Hill Institute both hold
conditions that expired in 2023). Publishing on that flag alone gives 145 providers "under a current
decision" instead of 112, i.e. an adverse claim about named businesses the register itself contradicts.
The rule is now *not historic AND not expired*, and a gate requires it to agree with the register's own
flag — 112 by our rule against 114 by the flag, 1.8% apart.

**One definition reversed on the evidence.** The research asserted suspended providers should count as
live supply. The verification showed the opposite: every suspended RTO carries a restriction record
saying the regulator has stopped it operating. Counting suspended scope as available training
overstates what a student can enrol in. Suspended is now excluded from teaching capacity (it does not
move the orphan count) but still counted as on the register.

## The sensitivity analysis

Rather than pick one definition and hope, the Method view publishes the headline recomputed five ways:

| Rule | Orphans |
|---|---|
| **No provider able to teach it (the site's rule)** | **144** |
| Counting assess-only providers (what training.gov.au's own search shows) | 137 |
| Counting suspended providers as able to teach | 144 |
| Only fully current providers | 149 |
| Excluding qualifications released in the last 12 months | 130 |

The spread is 130–149, which is the point: the finding does not depend on a flattering definition.
The 144 are also split by story — 94 never had a provider, 50 lost theirs, 14 are new enough that
nobody has applied yet, 2 are flagged confidential (so a zero may mean "withheld").

## Privacy

The organisation feed ships `ceo.{name,email,phone}` and the same again under `publicEnquiries` and
`registrationEnquiries`. This was confirmed on the very first record returned. Many RTOs are one- or
two-person businesses, so these are private mobile numbers and personal email addresses of real people.

- Every published record is built by an **explicit field allowlist**, never a denylist — a denylist is
  defeated the first time upstream renames a field, and the failure is silent and permanent in a public
  git history. A test injects an unknown contact field and asserts it is dropped.
- Regulator-authored prose is kept verbatim except that anything contact-shaped is redacted (it
  contains ASQA mailboxes).
- A **fatal gate** scans every generated file for email- and phone-shaped strings and forbidden keys.
  It fired for real during the build on agency mailboxes inside restriction text, which is exactly the
  behaviour wanted.

## Reconciliation Gates (13, all fatal)

| Gate | Result |
|---|---|
| Search harvest reconciles against the API's own totalCount | 13,146 = 13,146; 8,034 = 8,034 |
| Second ordered sweep at a different page size adds nothing | 0 added |
| Delivery pagination terminated for every qualification | 1,164 paged to exhaustion |
| Live filter discriminates (not all, not none) | 29,331 live of 39,182 rows |
| Pending/suspended counted correctly | MEM40322 (pending only) counted as 1 |
| Headline numerator and denominator are one universe | 1,164 = 1,164; 144 orphans all inside |
| Counts within fleet size | max 809, fleet 13,146 |
| Every delivery edge resolves to a known provider | all resolve |
| State coverage ≤ national for every qualification | 1,164 consistent |
| Decisions well formed and attached to known providers | 676 |
| Regulatory "in force" rule agrees with the register's flag | 112 vs 114, 1.8% apart |
| No personal data in any payload | 6 payloads clean |
| No NaN/Infinity/null-count in any payload | clean |

## Test Results
- Tests written: **174** across 5 files
- Tests passed: 174
- Tests failed: 0

`tests/gates.test.ts` (38) feeds each gate the exact failure it exists to catch — a truncated page, a
filter that removes nothing, a pending-only qualification called an orphan, a renamed status, a
mismatched denominator, a dangling edge, a stale isHistoric flag — and asserts the build stops.
`tests/parse.test.ts` (78) includes the privacy suite. `tests/overlay.test.ts` (15) asserts all four
dismissal exits by DOM state, not by "the handler ran". `tests/layout.test.ts` (18) asserts positions,
including pairwise non-overlap. `tests/hygiene.test.ts` (25) enforces the `[hidden]` guard, the
`.hover-tip.visible` rule, z-index ordering above Leaflet, `min-width: 0` on truncating children, no
GitHub links, and SPDX headers.

Four tests failed on first run. Three were **test** bugs — two scanners reading their own explanatory
comments (the CSS comment documenting `overflow-x: hidden` and the SPDX header naming AGPL), and a
wrong `zoomViewBox` signature. Fixed in the scanners, not by deleting the explanations. The fourth was
real: the four copied pattern files had no SPDX headers.

## Build Status
- npm install: pass
- npm test: pass (174/174)
- tsc --noEmit: pass
- npm run build: pass — 34.8 KB initial JS / 12.5 KB gzip, map chunk code-split to 155 KB
- Local preview: pass

## Verification performed

- **All 9 views** render with no error state, no stuck loading, no `NaN`/`undefined`/`Infinity`, and
  every network request 200.
- **Zero horizontal overflow at 375px on all 9 views** (`scrollWidth === clientWidth === 375`), and
  with the drawer both open and closed — the drawer detaches fully (overlay host has 0 children).
- **The modal dismissal contract at 375px with the About modal opened over the Leaflet map**: baseline
  clean (0 leaking `[hidden]`), 44×44 hit area, and all four exits asserted independently by DOM
  state — ✕, mouse `pointerdown` on the scrim, Escape, and a real touch `pointerdown`+`pointerup`
  with no double-fire reopen. Body scroll released afterwards.
- **Overlay stacking over Leaflet**: three probe points over the map all return elements inside
  `#overlay-host` while the modal is open. Max Leaflet z-index 1000, scrim 2000, panel 2100.
  Confirmed by screenshot, not only assertion.
- **Real trusted clicks** (never `element.click()`), after calibrating the screenshot→CSS ratio at
  1.600 with a `pointerdown` probe: a table row opened the dossier for ACM30921 *Certificate III in
  Equine Hoof Care* (a farrier qualification with zero providers in all eight jurisdictions); an SVG
  node opened AHC20724 and dimmed 346 of 347 nodes; a matrix cell opened *AHC · Graduate Diploma* →
  *Graduate Diploma of Arboriculture*, 0 providers. All set their deep link.
- **Drag-pan**: a drag pans the viewBox (x 171→263) **without** firing the node click; at zoom 1 it
  correctly does nothing because the viewBox equals the base.
- **Map**: 18 tiles, 8 real state polygons, tile-container transform is the identity matrix (no stuck
  zoom-animation scale), frame `isolation: isolate`.

## Deployment
- Repo created: yes — ben-gy/au-training
- GitHub Pages enabled: yes (build_type workflow)
- Cloudflare DNS CNAME created: yes — au-training → ben-gy.github.io
- TLS certificate: approved
- Directory entry live on main: yes
- Workflow triggered: yes

## Errors & Resolutions

| Problem | Resolution |
|---|---|
| Organisation harvest silently missing ~1,000 providers under default paging | `orderBy` for determinism + reconcile against totalCount + a second differently-sized sweep that must add nothing |
| `/delivery` truncating the headline qualification from 809 to 721 | Follow `next-page` to exhaustion; refuse an exact multiple of the page size with no terminating page |
| `Type/Id eq 1` returned 19,440 rows, not the 259 training packages assumed | Packages derived from the qualifications' own inline `{code,title}` instead — cheaper and guaranteed consistent |
| PII gate fired on ASQA mailboxes inside restriction text | Restriction prose routed through the same redaction as decision detail |
| "440 have five or fewer" contradicted the headline on the same screen | Both populations (441 one-to-five, 585 five-or-fewer) named explicitly |
| `liveEdges` counted assess-only scope the site's own rule excludes | Split into 29,056 teaching edges + 183 assess-only, from one predicate |
| Network spine packed 79 labels into ~9px each, colliding into a smear | Canvas height scales with the package (capped), and labels are suppressed below 13px pitch except the orphans and the most-supplied |
| Screenshot channel intermittently returned blank/mis-composited frames | Programmatic DOM assertions used as primary evidence; screenshots re-taken after scrolling to top |

## Notes for next time

- **Air quality is the strongest unbuilt idea** and is fully researched: NSW (137 stations, hourly to
  2010, no CORS so pipeline-only), QLD (118 stations live with `Access-Control-Allow-Origin: *`, plus
  718 historical CSVs, ~6M hourly rows) and ACT (383,694 rows, live within the hour, CORS `*`), all
  CC BY. Victoria, WA, SA and Tasmania are not obtainable. It would be a genuine hybrid: client-side
  realtime for QLD+ACT, pipeline for NSW and all history. The trap list is long and documented in the
  research (NSW returns HTTP 200 with an empty array for an invalid parameter triple; QLD silently
  ignores `limit` and caps at 1,000; QLD live rows are 100% unvalidated while the annual CSVs are the
  validated record).
- **Union-until-dry is a real technique** but `orderBy` is better where the API supports it: it is
  cheaper and provably complete rather than merely converged. Check for a sort parameter before
  reaching for the union.
- The NTR API also accepts OData `filter=`, so most headline counts are available as a single
  authoritative `totalCount` with no harvest at all — useful as an independent gate.
