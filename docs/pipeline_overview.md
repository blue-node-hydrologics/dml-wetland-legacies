# Pipeline overview

This document walks through the five workflow stages described in the paper
*"Drained Prairie Pothole Wetlands Provide Blueprints for Restoration in the
US Corn Belt"* (Van Meter & Basu, in prep) and points to the cells in each
notebook that implement them.

All notebooks read configurable data locations from environment variables
documented in the [README](../README.md): `DML_PROJECT_ROOT`,
`DML_GIS_DATA_ROOT`, `DML_NDVI_DATA_ROOT`, and `DML_GDRIVE_ROOT`.

## Stage 1 — Scene selection

**Notebook:** `notebooks/01_scene_selection/01_landsat_retrieval.ipynb`

Queries the USGS LandsatLook STAC catalog (proxied through Microsoft
Planetary Computer) for U.S. Landsat Collection 2 ARD Surface Reflectance
scenes covering the Des Moines Lobe by tile and date range. Scenes are
filtered by cloud cover and downloaded in parallel using a thread pool with a
retrying HTTP session. Outputs are organized on disk by year, month, and
tile under `$DML_PROJECT_ROOT/Landsat`.

**Key dependencies:** `pystac-client`, `planetary-computer`, `requests`,
`urllib3`, `tqdm`.

## Stage 2 — Monthly compositing

**Notebook:** `notebooks/02_main_pipeline/02_main_pipeline.ipynb`,
Step 0a (cells beginning with the heading *"Step 0a: Generate Monthly NDVI
Distributions"*) and the *NDVI Panel Max and Mosaic Generation* section.

For each ARD tile and each month, builds a maximum-NDVI composite from the
retrieved scenes and mosaics across the DML footprint. The composites are
the basis for all downstream zonal statistics.

**Key dependencies:** `rasterio` (including `rio_merge`, `WarpedVRT`,
`Resampling`), `numpy`, `concurrent.futures`.

## Stage 3 — Crop mask generation

**Notebook:** `notebooks/02_main_pipeline/02_main_pipeline.ipynb`,
Step 2 (cells beginning with the heading *"Step 2: Generate Annual Crop
Masks (2010–2024)"*).

Reads the USDA NASS Cropland Data Layer for each year, isolates corn (CDL
class 1) and soybean (CDL class 5), reprojects/aligns to the NDVI mosaics,
and writes annual binary masks. A small repair cell handles corrupted or
zero-byte CDL tiles.

**Key dependencies:** `rasterio`, `numpy`, `geopandas` (for the DML clip
polygon).

## Stage 4 — Wetland rasterization

**Notebooks:** `notebooks/03_wetland_layer_prep/`

- `03a_create_historical_wetland_layer.ipynb` — merges HUC8-level historical
  wetland shapefiles, repairs invalid geometries with `shapely.make_valid`,
  and clips to the DML.
- `03b_simplify_wetlands.ipynb` — applies a tolerance-based simplification to
  reduce vertex counts before rasterization.

The output is a single GeoPackage of historical wetland polygons in
EPSG:5070 that downstream zonal-statistics steps consume.

**Key dependencies:** `geopandas`, `shapely`, `pyproj`, `fiona`.

## Stage 5 — Zonal statistics

**Notebook:** `notebooks/02_main_pipeline/02_main_pipeline.ipynb`,
sections from *"Summarizing County-Level NDVI by Crop Type"* through
*"Crop-Specific Wetland vs. Adjacent Upland NDVI Analysis"*.

Two scales of zonal statistics are produced:

1. **County-level NDVI by crop type** — mean/median/std NDVI summarized by
   Iowa county for corn and soybean pixels separately, then linked to NASS
   county yield tables.
2. **Per-wetland NDVI** — for each historical wetland polygon, the same
   summary statistics are computed for pixels falling inside the wetland and
   in an adjacent upland buffer, enabling the wetland-vs-upland comparison
   that drives Figures 4, 5, 9, and S1.

**Key dependencies:** `rasterio`, `numpy`, `geopandas`, `pandas`.

## Supporting analyses (`notebooks/04_supporting_analyses/`)

- `04a_county_yield_analysis.ipynb` — peak-greenness vs NASS yield
  regressions.
- `04b_precipitation_analysis.ipynb` — gridded PRISM precipitation downloads
  and per-year summaries used for context.
- `04c_significance_historical_vs_upland.ipynb` — peak-greenness EVI/NDVI
  significance testing between historical wetlands and adjacent uplands.
- `04d_identify_sketched_farmland.ipynb` — identifies hand-sketched farm
  plots used as a sanity check on the per-wetland sampling.

## Figures (`notebooks/05_figures/`)

One notebook per paper figure. Each notebook reads the relevant outputs from
Stages 4 or 5 and renders the figure with `matplotlib`/`seaborn`. Filenames
prefixed with the figure number (`fig01_`, `fig03_`, …) sort in the order
they appear in the manuscript; notebooks prefixed with `fig_` produce
auxiliary panels and supporting visuals.
