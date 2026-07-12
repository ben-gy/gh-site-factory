# patterns/ — canonical visualization implementations

Proven, production-tested building blocks. **Copy these files into new sites and
adapt — re-rolling any of them from scratch is a defect.** Every one of them
exists because a hand-rolled variant shipped broken at least once.

| File | What it is | Provenance | How to use |
|---|---|---|---|
| `treemap.ts` | Squarified treemap layout `squarify(values, w, h): Rect[]` (Bruls et al.), pure geometry, NaN-guarded | nsw-pokies `src/charts.ts`, hardened in au-gov | Copy to `src/utils/squarify.ts`. Caller sorts values desc and applies origin offsets for nested levels. Ship `tests/layout.test.ts` with it. |
| `tooltip.ts` | Global `[data-tip]` hover tooltip: one floating div, delegated listeners, viewport-edge flipping | au-flights `src/tooltip.ts` | Copy to `src/components/tooltip.ts`, call `initTooltip()` once at boot, put `data-tip="…"` on every data mark (SVG or HTML). Style `.hover-tip` (fixed, high z-index, pointer-events none). Native SVG `<title>` is NOT a tooltip — use `aria-label` for a11y instead. |
| `network.ts` | Pure synchronous force layout: grid-bucketed repulsion, FR temperature cooling, deterministic (`mulberry32` seeding), time-budgeted | au-gov `src/utils/forceLayout.ts` | Copy to `src/utils/forceLayout.ts`. Run to completion BEFORE creating any DOM, then render final positions once — the graph must be motionless from the first frame. Handles 1,400 nodes in ~100–400 ms. Seed node positions clustered by group for faster settling. |
| `svgZoom.ts` | Wheel-zoom (about cursor) + drag-pan + zoom buttons + dblclick reset for any viewBox SVG; pure `zoomViewBox()` math exported for tests | au-gov `src/utils/svgZoom.ts` | Copy to `src/utils/svgZoom.ts`, call `attachSvgZoom(svg)` after rendering. Parent element needs `position: relative` for the `.svg-zoom-controls` buttons. Mandatory for dense SVG views (>~50 nodes). |
| `leafletMap.ts` | Annotated Leaflet template: CARTO basemap, real-GeoJSON layer, polygon + marker hover tooltips, attribution, zero-size defence | nsw-pokies `src/map.ts` + au-gov `src/views/map.ts` | Copy-adapt. Leaflet from npm (`leaflet@^1.9`), never CDN JS. |
| `geo/au-states.geojson` | Real ABS-derived AU state boundaries, `{code, name}` props | 2026-07-12-au-contracts | Copy into `public/data/`. See `geo/README.md` for sourcing anything else — never hand-author coordinates. |
| `tests/layout.test.ts` | Position-asserting layout test template: in-bounds, no-overlap, no-NaN, area conservation, degenerates | au-gov `tests/treemap.test.ts` | Copy next to any layout algorithm. Area-only tests pass on visually broken layouts. |

## Hard rules these encode

1. **Maps:** Leaflet + authoritative GeoJSON only (ABS/Geoscape/Natural Earth,
   mapshaper-simplified). Hand-authored coordinates in any format = fail.
2. **Force layouts settle before first paint.** No requestAnimationFrame layout
   animation, ever. Spatial grid for >~300 nodes.
3. **Every data mark has a styled hover tooltip** (data-tip util or Leaflet
   `bindTooltip`). Native `<title>` alone is a fail.
4. **Dense SVG views get zoom/pan** (`attachSvgZoom`).
5. **Nav tabs are words only** — no count badges in tab labels.
6. **Layout code ships positional tests** (no-overlap, in-bounds, no-NaN).
