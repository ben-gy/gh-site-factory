# Build Log: Aged Care Quality (AU)
**Date:** 2026-06-21
**Status:** deploy_failed (build complete, GitHub deploy blocked by broken `gh` auth)

## Idea Source
Researched. IDEAS.md was empty. After surveying registry.json for gaps in the AU site coverage (no aged care, childcare, schools, charities or contracts site yet), picked **residential aged care** because:
- A 2,596-service public dataset with rich multi-dimensional ratings is rare and highly actionable
- The decision is high-stakes for families and the official "Find a provider" tool is search-only, not comparable
- Government Star Ratings system gives 4 distinct rating dimensions plus 87 detail columns per service — perfect for multi-view product

## Site Details
- **Name:** Aged Care Quality (AU)
- **Repo:** ben-gy/au-agedcare (target — not yet pushed)
- **Category:** health
- **Audience:** families choosing aged care for an elderly parent; journalists, policy researchers, aged care managers
- **Stack:** Vanilla TypeScript + Vite 6 + Leaflet 1.9 + Vitest
- **Data strategy:** pipeline (XLSX → parsed → aggregated → JSON committed to repo)

## Data Sources
- Department of Health, Disability and Ageing — Star Ratings quarterly data extract (May 2026). Direct XLSX: https://www.health.gov.au/sites/default/files/2026-05/star-ratings-quarterly-data-extract-may-2026_0.xlsx (1.7 MB, 2,596 services × 87 columns)
- AIHW GEN aged care data — https://www.gen-agedcaredata.gov.au/resources/series (index of all historical extracts)
- Matthew Proctor — Australian Postcodes — https://github.com/matthewproctor/australianpostcodes (suburb centroids for the map view)

## Architecture Decisions
- **Vanilla TS, not React.** Eight panel views sharing a filter bar — a small set of imperative `render*` functions is simpler than a component tree with shared state. No router needed; URL hash handles the one piece of stateful linking (service detail).
- **Pipeline strategy.** Embedding 2,596 services × 87 columns directly in TS source would bloat the bundle and break TypeScript performance. JSON files served from `public/data/` and loaded at boot keep the bundle small (~60 KB gzipped) and the data isolated from compilation.
- **Leaflet + CARTO light tiles.** The data is geographic; a Leaflet map was the right call (NEVER hand-roll SVG maps — see CLAUDE memory). Light tiles fit the clinical/civic palette.
- **No chart library.** All charts (histograms, scatter, heatmap, matrix, RE bars) are hand-rolled SVG/divs — keeps the bundle lean and tooling-free.
- **Light theme, navy + teal + sage palette.** Audience is general public and civic, not technical. Dark/hacker aesthetic would be wrong (per CLAUDE memory).
- **Glossary tooltips and About modal mandatory.** Per CLAUDE memory — every domain term has a `glossary-link` span with click-to-show tooltip.
- **Sticky footer with `Built by benrichardson.dev` attribution.** No GitHub link. Per CLAUDE memory.

## Test Results
- Tests written: 32 (across `utils.test.ts`)
- Tests passed: 32
- Tests failed: 0

## Build Status
- npm install: pass
- npm test: pass (32/32)
- npm run build: pass (200 KB JS / 22 KB CSS, gzipped ~59 KB / ~4 KB)
- Local preview (`vite preview`): pass — verified via Claude Preview MCP, all 8 views rendered correctly, including the Leaflet map with 2,589 markers across Australia and the service detail panel via URL hash

## Deployment
- Repo created on github.com/ben-gy: **NO** — `gh repo create` failed with HTTP 401 (token invalid)
- GitHub Pages enabled: NO
- DNS / Cloudflare CNAME provisioned: NO
- PR created: NO
- Local git committed: YES — `main` branch, initial commit `7e4ecaf`

## Errors & Resolutions
- **`gh` CLI authentication broken** — `gh auth status` reports the token in the `ben-gy` user is invalid. SSH alternative not available (`~/.ssh/` has only `config`, no keys; `ssh git@github.com` fails host-key verification). Cannot push to GitHub from this run.
- **xlsx ESM import** — `import * as XLSX from 'xlsx'; XLSX.readFile(...)` failed because `readFile` requires Node's fs synchronously. Switched to `import { read as xlsxRead, utils as xlsxUtils } from 'xlsx'` + manual `readFileSync(buf)` + `xlsxRead(buf, { type: 'buffer' })`. Works.
- **Map tiles + markers not loading** — `requestAnimationFrame` inside `renderMap` was being throttled because the browser tab is backgrounded during MCP-driven testing. Replaced with `setTimeout(..., 0)`. Now reliable in both foregrounded and backgrounded tabs.
- **States histogram min-height** — `.histogram-bar { min-height: 4px }` made 0-count bars look like real data. Removed the CSS rule and gated `Math.max(2, …)` to data-driven heights only when `v > 0`.

## Manual follow-up needed
The build artefact, tests, and local git history are all present at `sites/2026-06-21-au-agedcare/`. To complete the deploy after re-authenticating `gh`:

```bash
cd /Users/benrichardson/Code/gh-site-factory/sites/2026-06-21-au-agedcare
gh auth login            # re-authenticate
gh repo create ben-gy/au-agedcare --private --source=. --push
gh api repos/ben-gy/au-agedcare/pages -X POST -f build_type=workflow
# Cloudflare CNAME + cert cycle as documented in SKILL.md step 9.
```

Once the site is live, change `status` in `registry.json` to `"deployed"` and add the entry to `index/sites.json` + `index/sites.txt` (skipped here because the site isn't reachable yet).
