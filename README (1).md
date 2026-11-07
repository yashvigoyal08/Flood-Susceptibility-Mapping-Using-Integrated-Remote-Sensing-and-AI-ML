# Flood Susceptibility Mapping in Paschim Medinipur, India Using Integrated Remote Sensing and AI/ML

A ten-year (2016–2025) Sentinel-1 flood record is combined with 13 terrain, hydrological, rainfall and vegetation factors to train Random Forest, XGBoost and LightGBM models that predict flood hazard in Paschim Medinipur district, West Bengal, India.

**Author:** Yashvi, Department of Agricultural and Food Engineering, IIT Kharagpur (Land and Water Resources Engineering)

---

## 1. Overview

Floods repeatedly damage farmland, settlements and infrastructure in Paschim Medinipur, yet fine-scale flood hazard information is limited. This project:

1. Builds a **10-year flood layer (2016–2025)** from Sentinel-1 SAR imagery on Google Earth Engine (GEE), using regionally calibrated backscatter thresholds.
2. Uses flood and non-flood samples from this layer to train **three tree-based machine learning models** (Random Forest, XGBoost, LightGBM).
3. Evaluates the models on a **spatially blocked test set**, so that scores reflect prediction in areas the models have not seen.

## 2. Workflow

```
Sentinel-1 (pre / monsoon / post)  ->  threshold-based flood detection per year (2016-2025)
            |                                           |
            |                        permanent water and rivers removed
            |                                           |
            +---------->  10-year flood layer (years flooded)  <----------+
                                        |
                  flood / non-flood samples + 13 conditioning factors
                                        |
                 Random Forest | XGBoost | LightGBM  (spatial block test)
                                        |
                            flood susceptibility maps
```

Flowchart images of the classification are in the `figures/` folder.

## 3. Study area

Paschim Medinipur district, West Bengal, India. The district boundary is a village-level polygon asset in Google Earth Engine (`projects/empyrean-bridge-495405-m8/assets/pachimi_medinipur`).

## 4. Data

| Data | Source | Use |
|---|---|---|
| Sentinel-1 GRD (IW, VV + VH) | `COPERNICUS/S1_GRD` | Flood detection |
| Sentinel-2 SR / TOA | `COPERNICUS/S2_SR_HARMONIZED`, `COPERNICUS/S2_HARMONIZED` | MNDWI and NDVI used in the GEE scripts (see notes) |
| ESA WorldCover 2021 | `ESA/WorldCover/v200/2021` | Cropland mask (class 40) |
| SRTM DEM | `USGS/SRTMGL1_003` | Slope mask |
| JRC Global Surface Water | `JRC/GSW1_4/GlobalSurfaceWater` | River and water-body mask |

## 5. Flood layer (Google Earth Engine)

### Time windows (each year)

| Period | Dates | Composite |
|---|---|---|
| Pre-monsoon | 1 Mar – 31 May | Median |
| Monsoon | 15 Aug – 30 Sep | 10th percentile (close to peak flood) |
| Post-monsoon | 15 Oct – 30 Nov | Median |
| Kharif window (NDVI) | 15 Aug – 30 Nov | 90th percentile NDVI |

A 30 m focal median is applied to each Sentinel-1 image for speckle reduction.

### Thresholds

| Parameter | Value |
|---|---|
| Open water, VV backscatter | < -17 dB |
| Open water, VH backscatter | < -24 dB |
| Permanent water confirmation, MNDWI | > 0 |
| Terrain slope for flood | < 5 degrees |
| Paddy, kharif NDVI (p90) | >= 0.6 |
| Paddy, pre-monsoon NDVI (p10) | < 0.3 |
| JRC water occurrence (river mask) | >= 10 % |
| JRC mask buffer | 60 m |

### Classification rules

| Class | Rule |
|---|---|
| Permanent water / rivers | Water in pre-monsoon AND post-monsoon (and MNDWI > 0), OR inside the JRC river mask |
| Flood extent | Water in monsoon AND NOT water in pre-monsoon AND NOT permanent water AND slope < 5 degrees |
| Paddy field | WorldCover cropland AND NDVI thresholds above AND NOT permanent water |
| Flooded paddy field | Flood extent AND paddy |
| Flooded area (non-paddy) | Flood extent AND NOT paddy |
| Non-flooded area | All other pixels |

### Multi-year flood layer

The yearly flood maps for 2016–2025 are computed in the background and added up to give the **number of years each pixel was flooded**. The final map script shows pixels flooded in at least `MIN_YEARS` years (default 2) and exports the result as a two-band image: years flooded, and a binary flood point layer.

## 6. Modelling

### Algorithms

Random Forest, XGBoost and LightGBM, three tree-based models that capture non-linear relationships and interactions without requiring data scaling.

### Conditioning factors (13)

| Group | Factors |
|---|---|
| Topographic | Elevation, slope, topographic position index (TPI), terrain ruggedness index (TRI) |
| Hydrological | Topographic wetness index (TWI), stream power index (SPI), upslope area, drainage density, distance to river |
| Climatic | Annual rainfall, extreme rainfall |
| Surface and accessibility | NDVI, distance to main road |

Multicollinearity was checked with the variance inflation factor (VIF); all factors had VIF below 3.

### Validation

Models were tested on a spatially blocked test set of 8,514 samples (4,041 non-flood and 4,473 flood), in which whole geographic blocks are held out from training.

## 7. Results

| Model | Accuracy | ROC-AUC | F1 (non-flood / flood) |
|---|---|---|---|
| Random Forest | 0.8807 | 0.9500 | 0.88 / 0.88 |
| LightGBM | 0.8789 | 0.9503 | 0.87 / 0.88 |
| XGBoost | 0.8642 | 0.9420 | 0.86 / 0.87 |

Random Forest and LightGBM performed almost identically; XGBoost was slightly lower.

### Comparison of the Sentinel-1 flood layer with the NRSC flood hazard atlas

The Sentinel-1 flood layer was compared with the NRSC flood hazard atlas at village level (5,714 villages): 85.3 % exact class agreement (79.0 % expected by chance), 98.0 % agreement within one class, and a quadratic weighted kappa of 0.497. Agreement was strongest in low-hazard areas and more moderate in the higher classes.

## 8. Repository contents

```
.
├── README.md
├── gee_scripts/
│   ├── flood_map_2025_pachimi_medinipur_v3.js          # single-year (2025) classification
│   ├── flood_map_2016_2024_pachimi_medinipur.js        # year-by-year maps with a year selector
│   └── flood_final_map_2016_2025_pachimi_medinipur.js  # 10-year background run + final map
├── figures/
│   ├── methodology_flowchart_flood_map_2025.png
│   └── threshold_based_flowchart_flood_map_2025.png
└── modelling/                                          # add model notebooks / scripts here
```

## 9. How to run

1. Open the [Google Earth Engine Code Editor](https://code.earthengine.google.com/).
2. Upload your district boundary as an asset and set its path at the top of the script (`aoi = ee.FeatureCollection(...)`).
3. Paste `flood_final_map_2016_2025_pachimi_medinipur.js` into a new script and click **Run**.
4. Check the Console for scene counts per year, then start the export tasks in the **Tasks** tab. The exports are written at 30 m in EPSG:32645.
5. If the map display is slow, run the asset export and load that asset for a fast display.

