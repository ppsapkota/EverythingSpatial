# Exposure Analysis — Papua New Guinea

Single-file MapLibre app that estimates **population and building exposure** for a
user-chosen area, broken down by **district (admin2, 87)** or **LLG (admin3, 326)** —
switchable at runtime. All statistics are computed **live in the browser** — no backend.

## What it does

- Click a point + set a radius (1–150 km), **or** draw a polygon.
- Choose the **breakdown level** (admin2 District / admin3 LLG) from the sidebar.
- Reports **people exposed** (WorldPop 100 m) and **buildings exposed** (Overture),
  as totals, as a share of the national figure, and split **by the chosen admin level**.
- Population is a true zonal sum of the WorldPop raster; the browser reads only the
  small window of pixels around the query via `geotiff.js`, so memory stays tiny even
  though the raster is 293 Mpx.
- Per-district split uses an **admin2-index raster aligned to the WorldPop grid**
  (`png_adm2_index.tif`): each population pixel already knows its district, so no
  per-pixel polygon test is needed.
- Buildings carry their `adm2_pcode` (spatial-joined in the pipeline), so the district
  breakdown and per-district `%` are exact group-bys.
- District/LLG table: sortable by people or buildings, each row shows its parent area,
  and exports to CSV.
- Buildings render as a density **heatmap** at national/provincial zoom (keeps the
  choropleth readable) and fade into individual **points** as you zoom in.
- Keyboard-operable controls, WCAG-AA text contrast, streamed load progress, inline
  coordinate validation, and a friendly empty-result state.

## Run locally

Served over `http://` (the app `fetch()`es its data; a `file://` double-click won't
load it). From the project root (`EverythingSpatial/`):

```bash
python -m http.server 8991
```

Then open <http://127.0.0.1:8991/app/exposure_analysis/>.

## Data layers (in `app/exposure_analysis/data/`, all `png_`-prefixed)

| File | Size | What |
|---|---|---|
| `png_admin0/1/2/3.geojson` | ~2–3 MB ea. | Country / 22 provinces / 87 districts / 326 LLGs (COD-AB). admin1/2/3 carry `pop_total` + `bld_total` + `idx`. |
| `png_pop_2026_100m.tif` | 14 MB | WorldPop 2026 100 m constrained. Shipped as-is (512-tiled → windowed decode in-browser). |
| `png_adm2_index.tif` / `png_adm3_index.tif` | ~3 MB ea. | uint16 area index aligned to the WorldPop grid (65535 = none), one per level. |
| `png_buildings.geojson` | **92 MB** | All 697,800 PNG building centroids. props: `a2` (adm2 pcode), `a3` (adm3 pcode), `h` height, `fl` floors, `st` subtype, `cl` class, `nm` name. |
| `png_meta.json` | 12 KB | District index↔pcode↔name, pop-raster georeference, provenance. |

## Regenerate

```bash
python pipeline/fetch_overture_buildings.py     # Overture → raw polygons + tagged points
python pipeline/make_exposure_layers.py         # → app/exposure_analysis/data/png_*
```

`fetch_overture_buildings.py` writes the raw building **polygons** (full attributes,
for QGIS) to `data/01_raw/buildings/` and a district-tagged **points** parquet to
`data/00_landing/buildings/`. `make_exposure_layers.py` turns landing inputs into the
`png_`-prefixed app layers above. Run stages selectively with `--only {admin,pop,index,buildings,meta}`
or `--skip-buildings`.

## Notes & caveats

- **81 MB buildings GeoJSON.** Shipped in full at the user's request. It loads and
  parses in a few seconds on localhost; the map clusters the points (toggle "All
  buildings"). For GitHub Pages this file must be committed un-LFS'd and is large —
  a future option is PMTiles for display while keeping the same in-browser count path.
- **Overture attribute coverage is sparse in PNG**: height 1.9 %, floors 3.2 %,
  type ~7 %, name 0.8 %. Most buildings are geometry-only; popups show "—" where absent.
- **Validation**: national population sums to **10,665,215** (matches ~10.7 M for PNG);
  Port Moresby (NCD) 10 km ≈ 504 k people / 12.8 k buildings.

## Extending to other countries

Everything is keyed off `CC = "png"` (a constant in both `index.html` and
`make_exposure_layers.py`) and the `png_`-prefixed filenames. Adding a country means
running the two pipeline scripts for its admin/WorldPop/Overture inputs and pointing
`CC`/`DATA` at the new prefix — the compute engine is country-agnostic.
