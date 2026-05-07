# Drained Prairie Pothole Wetlands Provide Blueprints for Restoration in the US Corn Belt

Code accompanying:

> **K.J. Van Meter**¹·², **N.B. Basu**³·⁴, and **D. Scharton**⁵ (*in prep, target: Nature Sustainability*).
> *Drained Prairie Pothole Wetlands Provide Blueprints for Restoration in the US Corn Belt.*

This repository contains the full analysis pipeline used to characterize the
performance of row-crop agriculture on historical (drained) wetlands of the
Des Moines Lobe (DML) physiographic region of the U.S. Corn Belt, using
Landsat surface reflectance, USDA Cropland Data Layer (CDL) crop masks, NASA
NASS county yield records, and a curated layer of historical wetland
polygons.

## Pipeline overview

The five workflow stages described in the paper are organized as follows:

| Stage | Description | Notebook |
| --- | --- | --- |
| 1. Scene selection | Query the USGS LandsatLook / Microsoft Planetary Computer STAC API for Landsat Collection 2 ARD Surface Reflectance scenes covering the DML by date and tile. | [`01_scene_selection/01_landsat_retrieval.ipynb`](notebooks/01_scene_selection/01_landsat_retrieval.ipynb) |
| 2. Monthly compositing | Build per-tile, per-month maximum-NDVI composites from the retrieved scenes and mosaic across the DML footprint. | [`02_main_pipeline/02_main_pipeline.ipynb`](notebooks/02_main_pipeline/02_main_pipeline.ipynb) (Step 0a) |
| 3. Crop mask generation | Generate annual corn and soybean masks (2010–2024) from CDL, aligned to the NDVI composites. | [`02_main_pipeline/02_main_pipeline.ipynb`](notebooks/02_main_pipeline/02_main_pipeline.ipynb) (Step 2) |
| 4. Wetland rasterization | Merge HUC8-level historical wetland shapefiles, repair geometries, simplify, clip to the DML, and rasterize. | [`03_wetland_layer_prep/`](notebooks/03_wetland_layer_prep/) |
| 5. Zonal statistics | Compute county-level and per-wetland NDVI statistics (mean, median, std) by crop type, and link to NASS county yields. | [`02_main_pipeline/02_main_pipeline.ipynb`](notebooks/02_main_pipeline/02_main_pipeline.ipynb) (Steps following Step 2) |

Supporting analyses (precipitation, peak-greenness significance tests,
sketched-farmland identification) live in `notebooks/04_supporting_analyses/`,
and figure-producing notebooks live in `notebooks/05_figures/`.

## Repository layout

```
.
├── notebooks/
│   ├── 01_scene_selection/
│   │   └── 01_landsat_retrieval.ipynb
│   ├── 02_main_pipeline/
│   │   └── 02_main_pipeline.ipynb
│   ├── 03_wetland_layer_prep/
│   │   ├── 03a_create_historical_wetland_layer.ipynb
│   │   └── 03b_simplify_wetlands.ipynb
│   ├── 04_supporting_analyses/
│   │   ├── 04a_county_yield_analysis.ipynb
│   │   ├── 04b_precipitation_analysis.ipynb
│   │   ├── 04c_significance_historical_vs_upland.ipynb
│   │   └── 04d_identify_sketched_farmland.ipynb
│   └── 05_figures/
│       ├── fig01_study_area.ipynb
│       ├── fig03_corn_soy_ndvi_regime_curves.ipynb
│       ├── fig04_ndvi_histograms_hist_vs_adjacent.ipynb
│       ├── fig05_uai.ipynb
│       ├── fig09_wetland_persistence_restoration_priority.ipynb
│       ├── figS1_crop_yield_vs_vi.ipynb
│       ├── fig_lollipop_plot.ipynb
│       ├── fig_probability_of_wetland_underperformance.ipynb
│       └── fig_restoration_potential_wetland_maps.ipynb
├── docs/
│   └── pipeline_overview.md      Stage-by-stage walkthrough
├── environment.yml               Conda environment (recommended)
├── requirements.txt              Pip fallback
├── .gitignore                    Ignores data, outputs, secrets
├── LICENSE                       MIT
├── CITATION.cff                  Machine-readable citation
└── README.md                     (this file)
```

## Installation

All analyses were developed and run in **Python 3.12** on macOS with the
geospatial stack installed via `conda-forge`. The recommended setup:

```bash
conda env create -f environment.yml
conda activate dml-legacy-wetlands
jupyter lab
```

A pip-only install is also available via `requirements.txt`, but installing
GDAL, rasterio, geopandas, fiona, and pyproj from PyPI requires a working
system GDAL. Most users will find conda dramatically easier.

The paper's named libraries are **rasterio**, **GDAL**, **geopandas**,
**numpy**, and **pandas**; additional dependencies (matplotlib, seaborn,
scipy, statsmodels, shapely, pyproj, contextily, libpysal/esda for spatial
statistics, scikit-image, opencv, pystac-client, planetary-computer, dbfread,
tqdm, requests) are pinned in the environment files.

## Data and configurable paths

This repository contains **code only**. The input datasets used by the
pipeline are publicly available but too large to redistribute here:

- **Landsat Collection 2 ARD Surface Reflectance** — USGS, accessed via the
  Microsoft Planetary Computer STAC endpoint
  (`https://planetarycomputer.microsoft.com/api/stac/v1`).
- **USDA NASS Cropland Data Layer (CDL)** — annual 30 m crop-type rasters,
  2010–2024.
- **USDA NASS county-level yield records** — corn and soybean, 2010–2024.
- **Historical wetlands of the Prairie Pothole Region** — HUC8-level
  shapefiles (Lane et al.; see paper for citation).
- **National Wetlands Inventory (NWI)** — current wetlands for comparison.
- **County boundaries** — TIGER/Line shapefiles, U.S. Census Bureau.

The notebooks read input and write output paths from **environment
variables**, so you do not need to edit notebook source to run the pipeline
on your machine. Set the variables once in your shell (or persist them in
`~/.zshrc`, `~/.bashrc`, or a project `.envrc`):

| Variable | What it points at | Default if unset |
| --- | --- | --- |
| `DML_PROJECT_ROOT` | Root of the project directory holding raw inputs and intermediate outputs (Landsat panels, CDL crop masks, wetland layers, NDVI summaries). | `.` (current directory) |
| `DML_GIS_DATA_ROOT` | Root of an external GIS-data directory (NWI geopackages, PPR boundary shapefile, CDL national mosaics). | `./gis_data` |
| `DML_NDVI_DATA_ROOT` | Root of the NDVI/EVI2 pooled per-wetland CSV outputs (originally lived on an external SSD). | `./ndvi_wetlands_data` |
| `DML_PPR_VECT_ROOT` | Root directory containing the HUC8-level Prairie Pothole Region vector outputs (used in `03_wetland_layer_prep/`). | `./ppr_vector_data` |
| `DML_GDRIVE_ROOT` | Root of a Google Drive mirror used for a couple of figure notebooks. | `./gdrive` |

For example:

```bash
export DML_PROJECT_ROOT="$HOME/data/NASA_UMRB_Legacy_Wetlands"
export DML_GIS_DATA_ROOT="$HOME/data/GIS_Data"
export DML_NDVI_DATA_ROOT="$HOME/data/ndvi_wetlands_data"
```

## How to reproduce

The notebooks are intended to be run roughly in the order suggested by the
folder numbering. A representative pass through the pipeline:

1. **Set the env vars** above so the notebooks can find your inputs and
   outputs.
2. **Pull Landsat scenes.** Run `01_scene_selection/01_landsat_retrieval.ipynb`
   to download ARD tiles for the years and months of interest.
3. **Prepare the historical wetland layer.** Run the notebooks in
   `03_wetland_layer_prep/` to merge, repair, simplify, and rasterize the
   wetland polygons.
4. **Run the main pipeline.** Open
   `02_main_pipeline/02_main_pipeline.ipynb` and execute cells in order.
   This produces monthly NDVI composites, annual crop masks, and the
   county-level and per-wetland NDVI summary tables.
5. **Run supporting analyses and figures** as needed
   (`04_supporting_analyses/`, `05_figures/`).

Cell-by-cell narration of each stage lives in
[`docs/pipeline_overview.md`](docs/pipeline_overview.md).

## Author affiliations

¹ Department of Geography, Pennsylvania State University, University Park, PA
² Earth and Environmental Sciences Institute, Pennsylvania State University, University Park, PA
³ Department of Earth and Environmental Sciences, University of Waterloo, Waterloo, Ontario
⁴ Department of Civil and Environmental Engineering, University of Waterloo, Waterloo, Ontario
⁵ Department of Environmental Systems, University of California, Merced, Merced, California

## License

Released under the MIT License — see [`LICENSE`](LICENSE).

## Citation

If you use this code, please cite the accompanying paper. A machine-readable
citation is provided in [`CITATION.cff`](CITATION.cff).

## Contact

Kimberly Van Meter — `vanmeterlab@gmail.com`
[github.com/blue-node-hydrologics/dml-wetland-legacies](https://github.com/blue-node-hydrologics/dml-wetland-legacies)
