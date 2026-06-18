# Montana Wildfire Detection & Risk Assessment Pipeline

> A cloud-native Earth Observation pipeline for wildfire risk modeling, active fire detection,
> and burn severity mapping across the state of Montana using Sentinel-2 imagery and machine learning.

---

## Overview

Wildfires are an increasingly critical threat to Montana's ecosystems, communities, and public lands.
This project builds an end-to-end geospatial data science pipeline that addresses two distinct but
complementary problems:

1. **Wildfire Risk Modeling** — Where is fire most likely to ignite and spread, *before* it starts?
2. **Active Fire & Burn Perimeter Detection** — Once a fire starts, can we detect it and define its
   perimeter from satellite imagery in near real-time?

The pipeline is built on cloud-native EO principles using Sentinel-2 multispectral imagery accessed
via STAC, processed with Open Data Cube tooling, and analyzed using a combination of spectral indices
and machine learning models. Historical fire perimeter data from NIFC and MTBS provides the ground
truth for model training and validation.

The geographic scope is the **entire state of Montana**, covering diverse ecosystems including the
Rocky Mountain Front, the Northern Rockies, the Columbia Plateau, and the Great Plains — each with
distinct fire behavior characteristics.

---

## Project Goals

- Build a reproducible, cloud-native EO data pipeline using `pystac-client`, `odc-stac`, and `xarray`
- Compute meaningful spectral indices from Sentinel-2 imagery relevant to fire risk and burn severity
- Train and evaluate machine learning models for:
  - **Pre-fire risk scoring** using historical fire occurrence, vegetation, topography, and climate variables
  - **Active fire and burn perimeter detection** using multi-temporal Sentinel-2 change detection
- Produce analysis-ready raster outputs and visualizations covering the state of Montana
- Demonstrate best practices in geospatial data engineering, remote sensing analysis, and reproducible science

---

## Repository Structure

```
montana-wildfire-pipeline/
│
├── data/                        # Local data cache (gitignored)
│   ├── raw/
│   └── processed/
│
├── notebooks/                   # Exploratory analysis and demonstrations
│   ├── 01_stac_data_access.ipynb
│   ├── 02_spectral_indices.ipynb
│   ├── 03_risk_model_training.ipynb
│   ├── 04_fire_detection.ipynb
│   └── 05_results_visualization.ipynb
│
├── pipeline/                    # Core pipeline modules
│   ├── __init__.py
│   ├── stac_query.py            # STAC search and scene selection
│   ├── preprocessing.py         # Cloud masking, band math, normalization
│   ├── indices.py               # Spectral index computation
│   ├── risk_model.py            # Pre-fire risk scoring model
│   ├── fire_detection.py        # Active fire and perimeter detection
│   └── utils.py
│
├── models/                      # Saved model artifacts
│
├── outputs/                     # Generated maps and rasters (gitignored)
│
├── configs/
│   └── montana_aoi.geojson      # State of Montana AOI boundary
│
├── tests/
│
├── environment.yml              # Conda environment
├── requirements.txt
└── README.md
```

---

## Module 1 — Data Pipeline

### Data Sources

| Dataset | Source | Purpose |
|---|---|---|
| Sentinel-2 L2A | Element84 / AWS STAC | Primary EO imagery |
| MTBS Fire Perimeters | USGS MTBS | Burn severity ground truth |
| NIFC Historical Perimeters | NIFC GeoMAC | Historical fire occurrence |
| NLCD Land Cover | USGS | Vegetation / fuel type |
| 3DEP DEM | USGS | Topography (slope, aspect, elevation) |
| GRIDMET Climate | University of Idaho | Climate / fire weather variables |

### STAC Access Pattern

Imagery is accessed cloud-natively via `pystac-client` against the Element84 Earth Search STAC
catalog, filtered by:

- Area of Interest: State of Montana (from AOI GeoJSON)
- Date range: configurable (defaults to current fire season)
- Cloud cover threshold: < 20%
- Sentinel-2 Collection: `sentinel-2-l2a`

Scenes are loaded lazily using `odc-stac` into `xarray` DataArrays with consistent CRS (EPSG:32612)
and spatial resolution (10m for visible/NIR, 20m for SWIR bands).

### Preprocessing

- Cloud and cloud shadow masking using the Sentinel-2 SCL band
- Temporal compositing (median) for cloud-free mosaics over defined periods
- Band scaling (DN to surface reflectance)
- Reprojection and alignment to a common grid

---

## Module 2 — Spectral Indices

The following spectral indices are computed from Sentinel-2 bands and form the core feature set
for both modeling tasks:

| Index | Formula | Purpose |
|---|---|---|
| **NDVI** | (B8 - B4) / (B8 + B4) | Vegetation density and health |
| **NBR** | (B8 - B12) / (B8 + B12) | Burn ratio; pre/post fire comparison |
| **dNBR** | NBR_pre - NBR_post | Delta burn ratio; burn severity classification |
| **NBR2** | (B11 - B12) / (B11 + B12) | Post-fire soil exposure |
| **NDWI** | (B3 - B8) / (B3 + B8) | Vegetation moisture content |
| **BSI** | ((B11 + B4) - (B8 + B2)) / ((B11 + B4) + (B8 + B2)) | Bare soil index |
| **NDMI** | (B8 - B11) / (B8 + B11) | Moisture index; fuel moisture proxy |

dNBR burn severity is classified using standard USGS thresholds into:
`Unburned → Low → Moderate-Low → Moderate-High → High Severity`

---

## Module 3 — Pre-Fire Risk Modeling

### Objective

Produce a continuous wildfire risk score layer for the state of Montana by learning from historical
fire occurrence patterns combined with static and dynamic environmental predictors.

### Feature Set

**Static features (topographic & land cover):**
- Elevation, slope, aspect (from 3DEP DEM)
- Land cover class (NLCD)
- Distance to roads, distance to water

**Dynamic features (seasonal / annual):**
- NDVI, NDWI, NDMI from Sentinel-2 composites
- GRIDMET fire weather variables (ERC, VPD, wind speed, relative humidity)
- Antecedent precipitation

### Approach

- Ground truth: historical NIFC fire perimeters (burned = 1, unburned = 0) with
  spatial sampling to address class imbalance
- Models evaluated: Random Forest, XGBoost, and optionally a simple spatial CNN
- Validation: spatial cross-validation (buffered to prevent data leakage across folds)
- Output: continuous risk score raster (0–1) for Montana at 100m resolution

---

## Module 4 — Active Fire & Burn Perimeter Detection

### Objective

Detect actively burning pixels and map evolving fire perimeters from multi-temporal
Sentinel-2 imagery during a fire event.

### Approach

**Spectral change detection:**
- Compute NBR and NDVI for a pre-fire baseline composite and active/post-fire scenes
- Identify burn signal using dNBR thresholding as a baseline method

**Machine learning detection:**
- Train a pixel-level classifier (Random Forest or XGBoost) on labeled burned/unburned
  pixels from MTBS-validated historical fire scenes
- Features include spectral indices, raw band values, and temporal difference features
- Apply classifier to new scenes for near real-time perimeter delineation

**Perimeter vectorization:**
- Convert classified burn rasters to vector polygons using `rasterio` and `shapely`
- Smooth and simplify perimeter geometry
- Output GeoJSON perimeters timestamped to scene acquisition date

### Candidate Validation Fires

The following Montana fires have good Sentinel-2 coverage and MTBS validation data:

- **Trail Creek Fire (2022)** — Near Missoula; well-documented
- **Elmo Fire** — Flathead Reservation; large perimeter
- **Glacier Loon Fire (2023)** — Glacier NP adjacency; complex terrain

---

## Results & Outputs

- **Risk score raster** — Statewide wildfire risk layer (GeoTIFF)
- **Burn severity maps** — Per-fire dNBR classified rasters
- **Detected fire perimeters** — GeoJSON polygons with acquisition timestamps
- **Interactive map** — Folium or Kepler.gl visualization of risk layer + historical perimeters
- **Model evaluation report** — Accuracy metrics, feature importance, spatial validation results

---

## Environment & Dependencies

```bash
conda env create -f environment.yml
conda activate wildfire-pipeline
```

Core dependencies:

- `pystac-client` — STAC catalog search
- `odc-stac` — Cloud-native Sentinel-2 loading
- `xarray` / `rioxarray` — Raster data handling
- `geopandas` / `shapely` — Vector operations
- `scikit-learn` / `xgboost` — Machine learning
- `rasterio` — Raster I/O and vectorization
- `folium` / `matplotlib` — Visualization
- `dask` — Parallel processing for statewide analysis

---

## Roadmap

- [ ] STAC query module and scene selection logic
- [ ] Cloud masking and preprocessing pipeline
- [ ] Spectral index computation and validation
- [ ] Risk model training and spatial cross-validation
- [ ] Active fire detection classifier
- [ ] Perimeter vectorization and smoothing
- [ ] Statewide risk raster generation
- [ ] Interactive visualization
- [ ] Documentation and notebook walkthroughs

---

## Author

Built by [Your Name] as a portfolio project demonstrating geospatial data engineering,
Earth Observation analysis, and applied machine learning for environmental applications.

*Missoula, Montana*

---

## License

MIT License
