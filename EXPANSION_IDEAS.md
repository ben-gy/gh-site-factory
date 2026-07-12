# Expansion Ideas

Ideas for improving existing sites with new data, views, or features. These are NOT new site ideas — they're enhancements to products we've already built. The scheduled task logs these here instead of building duplicate sites.

Review periodically and fold the best ones into the existing repos.

<!-- Format:
- **repo-name**: Description of the enhancement, new data source, or additional view. Include URLs for any new data sources.
-->

- **au-contracts**: The deployed site covers FY2023-24 + FY2024-25 (period 2023-07-01 → 2025-06-30) with a consulting-spend and tender-method focus. The just-completed **FY2025-26** (1 Jul 2025 – 30 Jun 2026) is now fully available from the AusTender OCDS API and is NOT in the site — the pipeline window is fixed and won't roll forward on its own. Enhancement: add FY2025-26 (roughly 57k contracts / ~$87B) so the year-over-year view runs through the newest year, and make the pipeline window advance automatically each 1 July (e.g. "most recent complete FY"). API: `https://api.tenders.gov.au/ocds/findByDates/contractPublished/{startISO}/{endISO}` — paginate via `links.next` cursor, dedupe amendments by base CN number (strip `-A\d+` suffix), classify by UNSPSC 2-digit segment. (A duplicate single-year FY2025-26 build was attempted on 2026-07-12 after a worktree recycle; folded here instead of shipping a second site.)
