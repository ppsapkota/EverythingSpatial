# Cyclone track viewer

Single-file MapLibre GL app for the IBTrACS best-track archive.

## Run locally

When served over `http://`, the app `fetch()`es the GeoJSON — opening `index.html`
as a `file://` double-click won't load the tracks this way (browsers block `file://`
fetches). For a double-clickable copy, build the portable package (below).

From the project root (`EverythingSpatial/`):

```bash
python -m http.server 8899
```

Then open <http://127.0.0.1:8899/app/>.

## Deploy (GitHub Pages)

The app is self-contained under `app/`: `index.html` loads its data via `./data/...`
relative paths, so publish **`app/` as the Pages root** and the page ships with its
data. Nothing under the repo's `data/` (raw/processed/curated) is published.

Checklist: `app/data/` must be committed (not `.gitignore`d), not tracked by Git LFS
(Pages serves LFS pointers, not content), and filename case must match exactly (Pages
is case-sensitive). GitHub serves the GeoJSON gzipped (~18 MB → a few MB on the wire).

## Portable / double-click package

For a copy that opens by double-clicking (no server), run
`python pipeline/make_portable_package.py` → `dist/cyclone_tracks/`. It loads data via
`<script>` tags into a registry (file:// can't `fetch()` siblings) and vendors MapLibre
locally; only the basemap tiles need internet. `index.html` auto-detects: registry if
present, otherwise fetch.

## Configure

Everything is driven by config blocks near the top of `index.html`:

- `DATA_URL` — path to the tracks GeoJSON.
- `BASEMAPS` — basemap options (OSM grey/light/dark + Mapbox). Default `osm-grey`
  (CARTO Positron). Mapbox needs a token, entered in the sidebar at runtime.
- `LAYERS` — the layer registry. **To add a layer, append one object** and pick a
  `type`; it renders and appears in the layer list automatically. Commented
  templates cover every supported type (points, polygons, heatmap, CSV/Google
  Sheet points, raster XYZ, WMS, vector tiles, PMTiles, image overlay).
- `CATS` / `SSHS_COLOR` — Saffir–Simpson colours, shared by the ramp, chips, legend.

## Data layers (medallion)

```
data/00_landing/cyclone_tracks/    transient drop zone (unused by the pipeline)
data/01_raw/cyclone_tracks/        original download: <dataset>_<ts>.zip + unzipped .shp
data/02_processed/cyclone_tracks/  cleaned archive + season subsets  <name>_<ts>.gpkg
data/03_curated/cyclone_tracks/    El Niño episode extracts          <product>_<ts>.gpkg
app/data/cyclone_tracks/           what the app loads                <product>.geojson
```

The app-delivery layer sits under `app/` (not `data/`) on purpose: the app folder
is then self-contained, so publishing `app/` as a GitHub Pages root ships the page
and its data together via `./data/...` relative paths, and the large raw/processed
layers under `data/` never get published. `paths.APP` points here.

Which script writes where: `preprocess_ibtracs.py` → 01_raw + 02_processed + data_app.
`make_elnino_layers.py` → 03_curated + data_app. A season-floored subset is still a
*processed* form of the source, so it stays in 02; 03 is for downstream analytical
products built on top of the processed archive.

`make_geoparquet.py` adds a GeoParquet sibling (same stem + stamp, `.parquet`)
next to every GeoPackage in 02/03 — ~8–12 % of the gpkg size (zstd, columnar),
QGIS-native, and much faster to read in pandas/geopandas. Run it after either
pipeline script. Parquet never goes to `data_app`: MapLibre has no Parquet
source type, so the browser keeps getting GeoJSON.

Run stamps are UTC, format `_20260721T0926Z`. Layers 01–03 carry them so every
derived file is traceable to its run and older versions remain available
(`prune_old` keeps the newest 3).

**`data_app` is deliberately unstamped** — `index.html` references those filenames
directly, so a changing name would break the app on every regeneration. The app
layer holds the current copy under a stable name; history lives in 02/03.

Formats differ by layer on purpose: GeoPackage is the analytical carrier for 02/03
(typed, compact, QGIS-native); GeoJSON is only a browser delivery format, so it
appears solely in `data_app`.

## Source

IBTrACS `since1980` (v04r01, lines) — seasons **1980–2026**.

The full archive is ~301,000 segments ≈ **137 MB** of GeoJSON, too big for the
browser and over GitHub's 100 MB per-file cap. So:

- **Full archive** → `data/data_app/IBTrACS.since1980.list.v04r01.lines.gpkg`
  (72 MB, all seasons) for QGIS/analysis.
- **Web slice** → a season-floored GeoJSON that the app loads. `DATA_URL` in
  `index.html` points at it.

| Slice | Segments | Storms | GeoJSON |
|---|---|---|---|
| `--min-season 2020` (current default) | 39,149 | 684 | 18 MB |
| `--min-season 2015` | 72,049 | 1,225 | 33 MB |
| full 1980–2026 | 301,388 | 4,885 | ~137 MB |

Regenerate:

```bash
python pipeline/preprocess_ibtracs.py --dataset since1980 --min-season 2015 --formats geojson
```

then update `DATA_URL`. For the *whole* archive in the browser, the next step is
PMTiles (tippecanoe → single static file, HTTP range requests) — the frontend
code survives that migration nearly untouched.

Note: recent seasons ship as `PROVISIONAL` / `US-PROVISIO` track types. The
pipeline keeps them (dropping only `spur-*` duplicates) and flags them via a
`PROVISIONAL` field, which the app surfaces as a badge.
