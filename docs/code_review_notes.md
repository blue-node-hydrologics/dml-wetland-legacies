# Code Review Notes
*dml-wetland-legacies — internal review, May 2026*

Overall the pipeline is well-structured, cleanly commented, and follows good practices (env-var-driven paths, retry logic, parallel downloads, explicit reference grids). The issues below are refinements, not rewrites — most can be fixed in under an hour.

---

## 🔴 High Priority — Fix Before Making Repo Public

### 1. Hardcoded machine-specific path in `03a_create_historical_wetland_layer.ipynb`

**What:** `SRC_DIR` is still set to an absolute path on your development machine:
```python
SRC_DIR = Path("/Volumes/GlenCanyon/files/Prairies_Fill_Sinks/.../UMRB_vectoroutputs")
```
Every other script in the repo correctly uses `os.environ.get("DML_...")` for external paths, but this one was missed.

**Fix:** Add a new env var (e.g., `DML_PPR_VECT_ROOT`) and update the line:
```python
SRC_DIR = Path(os.environ.get("DML_PPR_VECT_ROOT", "./ppr_vector_data"))
```
Then document `DML_PPR_VECT_ROOT` in the README env-vars table.

---

### 2. `!pip install` cell with verbose cached output in `01_landsat_retrieval.ipynb`

**What:** Cell 2 runs `! pip install pystac-client planetary-computer requests tqdm tenacity urllib3` and the notebook still contains the full cached output (~50 lines of "Requirement already satisfied"). This is fine during development but looks messy in a published repo.

**Fix (two steps):**
- Delete the `! pip install` cell entirely — installation should happen via `environment.yml`, which is the documented method.
- Run `jupyter nbconvert --clear-output --inplace notebooks/01_scene_selection/01_landsat_retrieval.ipynb` to clear the output.

---

### 3. Debug cell `1 + 1` in `01_landsat_retrieval.ipynb`

**What:** There is a lone `1 + 1` cell (cell b011382f) — a classic "is the kernel alive?" test left in by accident.

**Fix:** Delete the cell.

---

### 4. Duplicate analysis script in `02_main_pipeline.ipynb` (cells 17 & 18)

**What:** Both cells contain what appears to be the same script (`analyze_ndvi_wetland_upland_by_year_plus_metrics_monthly.py`), with cell 17 prefixed with the comment `#updated`. These are two versions of the same code kept side-by-side during development.

**Fix:** Delete the older version (cell 17 or 18, whichever is superseded). If both represent genuinely different things, rename them clearly.

---

### 5. Stub/empty cells in `02_main_pipeline.ipynb`

**What:** Several cells contain only a comment with no functional code:
- Cell 19: `# analysis`
- Cell 21: `# summarize annual wetland area and number`
- Cell 41: completely empty

**Fix:** Delete them, or flesh them out if they were placeholders for planned code.

---

### 6. Missing packages in `environment.yml`

Two packages used in the notebooks are not listed in `environment.yml`:

| Package | Used in | Fix |
|---------|---------|-----|
| `tenacity` | `01_landsat_retrieval.ipynb` (pip install cell) | Add `- tenacity` under dependencies |
| `pyogrio` | `03a_create_historical_wetland_layer.ipynb` (`gpd.options.io_engine = "pyogrio"`) | Add `- pyogrio` under dependencies |

Both are available on conda-forge. Without them, users who follow the documented `conda env create -f environment.yml` path will have an incomplete environment.

---

### 7. Author affiliation placeholders in `README.md`

**What:** Lines 149–152 contain:
```
¹ Department of Earth and Environmental Sciences, [Affiliation 1 — fill in]
² [Affiliation 2 — fill in]
```

**Fix:** Fill these in before making the repo public.

---

## 🟡 Medium Priority — Polish

### 8. Hardcoded paths leak into markdown cell descriptions

In `03a_create_historical_wetland_layer.ipynb`, the markdown description cell still mentions the development machine path (`/Volumes/GlenCanyon/...`) and the external drive path (`/Volumes/Conowingo/...`). The code has been updated to use env vars, but the prose description wasn't updated to match.

**Fix:** Update the markdown to say "path configured via `DML_PPR_VECT_ROOT`" and "path configured via `DML_NDVI_DATA_ROOT`" rather than the literal volume names.

---

### 9. Typo in `environment.yml`

Line 22: `# Geospatial stack (papers's named libraries first)` — should be `paper's`.

---

### 10. Double-mask pattern in `03b_simplify_wetlands.ipynb`

**What:** The geometry repair block evaluates the boolean mask twice:
```python
gdf.loc[~gdf.geometry.is_valid, "geometry"] = gdf.loc[~gdf.geometry.is_valid, "geometry"].buffer(0)
```
This recomputes `~gdf.geometry.is_valid` twice, which is fine but slightly wasteful on 200k+ features.

**Fix (minor):**
```python
mask = ~gdf.geometry.is_valid
if mask.any():
    gdf.loc[mask, "geometry"] = gdf.loc[mask, "geometry"].buffer(0)
```

---

### 11. `DML_PATH` uses wrong root variable in `03a`

**What:** The DML boundary shapefile path is constructed as:
```python
DML_PATH = Path(os.environ.get("DML_NDVI_DATA_ROOT", ...)) / "Shapefiles/dml_boundaries_5070.shp"
```
A DML boundary shapefile is GIS reference data, not NDVI output data — it logically belongs under `DML_GIS_DATA_ROOT`.

**Fix:**
```python
DML_PATH = Path(os.environ.get("DML_GIS_DATA_ROOT", "./gis_data")) / "Shapefiles/dml_boundaries_5070.shp"
```
Update the README table accordingly.

---

## 🟢 Low Priority / Nice to Have

### 12. No version pins in `environment.yml`

Unpinned environments are fine for active development, but for reproducibility (and reviewers trying to run the code in 2027), pinning major packages is good practice. At a minimum:
```yaml
- rasterio>=1.3
- geopandas>=1.0
- shapely>=2.0
- python=3.12
```
Shapely 2.0 in particular introduced breaking changes from 1.x; pinning `>=2.0` ensures users don't get the old API.

---

### 13. No `nbstripout` or CI

Consider adding `nbstripout` as a git pre-commit hook so cell outputs are never accidentally committed:
```bash
pip install nbstripout
nbstripout --install
```
For a published repo, even a minimal GitHub Actions workflow that checks the notebooks parse without error adds credibility:
```yaml
# .github/workflows/lint.yml
name: Lint notebooks
on: [push, pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install nbformat
      - run: python -c "import nbformat; [nbformat.read(f, as_version=4) for f in __import__('pathlib').Path('notebooks').rglob('*.ipynb')]"
```

---

### 14. `CITATION.cff` `notes` field

The `notes` field on a `preferred-citation` entry is non-standard CFF (the spec doesn't define it). It won't break anything, but validators may flag it. The standard way to mark an in-prep paper is to omit the `year` field until accepted, or use `status: "manuscript-in-preparation"` as a custom extension comment.

---

## Summary Checklist

| # | File | Action | Priority |
|---|------|---------|----------|
| 1 | 03a notebook | Replace hardcoded `SRC_DIR` with env var | 🔴 |
| 2 | 01 notebook | Remove `!pip install` cell + clear outputs | 🔴 |
| 3 | 01 notebook | Delete `1 + 1` debug cell | 🔴 |
| 4 | 02 notebook | Delete duplicate analysis script cell | 🔴 |
| 5 | 02 notebook | Delete stub cells (19, 21, 41) | 🔴 |
| 6 | environment.yml | Add `tenacity` and `pyogrio` | 🔴 |
| 7 | README.md | Fill in author affiliations | 🔴 |
| 8 | 03a notebook | Update markdown prose to reference env vars | 🟡 |
| 9 | environment.yml | Fix `papers's` typo | 🟡 |
| 10 | 03b notebook | Store boolean mask before double-use | 🟡 |
| 11 | 03a notebook | Use `DML_GIS_DATA_ROOT` for DML boundary path | 🟡 |
| 12 | environment.yml | Add minimum version pins for key packages | 🟢 |
| 13 | (new) | Add `nbstripout` and/or GitHub Actions lint | 🟢 |
| 14 | CITATION.cff | Clean up non-standard `notes` field | 🟢 |
