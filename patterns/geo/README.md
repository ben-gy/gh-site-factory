# Boundary GeoJSON — authoritative sources only

**NEVER hand-author map coordinates in any format** — not SVG paths, not GeoJSON,
not TypeScript arrays. A boundary file under ~50 KB, or a state/country polygon
with fewer than ~50 vertices, is an automatic fail (that's how the au-gov map
shipped with NT and the ACT as literal rectangles).

## Files in this directory

| File | Coverage | Source | Size |
|---|---|---|---|
| `au-states.geojson` | 8 AU states/territories, `properties: {code, name}` | ABS ASGS-derived (CC BY 4.0 — attribute as "Boundaries: ABS ASGS (CC BY 4.0)") | 395 KB |

Reuse these before downloading anything new. Copy into the site's `public/data/`.

## Getting new boundaries

- **Australia (states, LGAs, SA regions):** ABS ASGS digital boundary files, or
  Geoscape Administrative Boundaries via data.gov.au (both CC BY 4.0).
  Example: nsw-pokies downloads the Geoscape NSW LGA WFS GeoJSON in its pipeline.
- **World countries:** Natural Earth (public domain).

## Simplify with mapshaper (in the pipeline, never by hand)

```bash
npx mapshaper raw.geojson -simplify 5% keep-shapes -o precision=0.0001 out.geojson
# dissolve to one feature per region first if the source is parcel-level:
npx mapshaper raw.geojson -dissolve2 LGA_NAME -simplify 5% keep-shapes -o precision=0.0001 out.geojson
```

Target 100 KB–1 MB for the shipped file: real coastlines, fast load.
