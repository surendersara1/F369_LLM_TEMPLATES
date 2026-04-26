<!-- Template Version: 2.0 | F369 Wave 8 (composite) | Composes: MLOPS_GEOSPATIAL_ML + MLOPS_BATCH_TRANSFORM + MLOPS_SAGEMAKER_TRAINING -->

# Template 32 — Geospatial ML (Earth observation imagery · pre-built models · custom segmentation · raster ops)

## Purpose

Build a satellite/aerial imagery analytics pipeline using SageMaker Geospatial: Earth Observation Jobs (EOJ) pulling Sentinel-2 / Landsat from AWS Open Data, applying pre-built models (cloud removal, NDVI, building footprint, change detection) OR custom segmentation training. Includes batch inference on imagery archives + visualization via Geospatial Studio.

Generates production-deployable CDK + EOJ trigger Lambda + custom training pipeline (if needed) + visualization runbook.

---

## Role Definition

You are an expert AWS geospatial + ML engineer with deep expertise in:
- SageMaker Geospatial (region-pinned us-west-2)
- Earth Observation Jobs (input config, time range filters, AOI polygons, output config)
- Pre-built geospatial models (cloud removal, LULC, NDVI, building footprint, change detection)
- Sentinel-2 + Landsat imagery from AWS Open Data Registry
- Cloud Optimized GeoTIFF (COG) format
- Raster ops (rasterio, geopandas) on EMR Serverless
- Geospatial Studio visualization

Generate complete production-deployable code, no TODOs.

---

## Context and Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED — must be us-west-2 for Geospatial]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED — dev | stage | prod]

# --- USE CASE ---
USE_CASE:               [REQUIRED — crop_health | deforestation | urban_planning | disaster_assessment | custom]

# --- AOI ---
AOI_GEOJSON_S3_URI:     [REQUIRED — polygon GeoJSON]
DATE_RANGE_START:       [REQUIRED — YYYY-MM-DD]
DATE_RANGE_END:         [REQUIRED — YYYY-MM-DD]

# --- IMAGERY ---
IMAGERY_SOURCE:         [REQUIRED — sentinel2 | landsat | byop]
CLOUD_COVER_MAX:        [OPTIONAL — 20 (%)]

# --- MODEL TYPE ---
MODEL_TYPE:             [REQUIRED — cloud_removal | lulc | ndvi | building_footprint | change_detection | custom_segmentation]

# --- CUSTOM TRAINING (if MODEL_TYPE=custom_segmentation) ---
TRAINING_TILES_S3:      [if custom — labeled tiles bucket]
NUM_CLASSES:            [if custom]
INPUT_BANDS:            [if custom — comma-separated, e.g. red,green,blue,nir,swir]
ARCHITECTURE:           [if custom — deeplabv3plus | unet | maskrcnn]

# --- OUTPUT ---
OUTPUT_FORMAT:          [REQUIRED — geotiff | geojson | csv]
OUTPUT_RESOLUTION:      [OPTIONAL — high | medium | low]

# --- COMPLIANCE ---
KMS_KEY_ARN:            [REQUIRED]
```

---

## Partial Library

| Partial | Why |
|---|---|
| `MLOPS_GEOSPATIAL_ML` | EOJ creation + pre-built models + custom training + cost matrix |
| `MLOPS_BATCH_TRANSFORM` | For bulk inference on imagery archives |
| `MLOPS_SAGEMAKER_TRAINING` | Custom segmentation training pipeline |
| `LAYER_NETWORKING` | VPC config for cross-region copy back to home region |

---

## Architecture

```
   AWS Open Data Registry (Sentinel-2 / Landsat) — public, free
        │
        │  EOJ pulls scenes by AOI + date + cloud cover filter
        ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │  Earth Observation Job (us-west-2 only)                          │
   │     - Input: AOI polygon + 2026-04-01..15 + cloud<20%             │
   │     - Pre-built model: CloudRemoval | LULC | NDVI | Building...   │
   │     - Output: cleaned GeoTIFF per scene                           │
   └────────────────┬─────────────────────────────────────────────────┘
                    │
                    ▼
   S3 bucket (us-west-2): processed imagery (KMS-encrypted, COG format)
        │
        ▼ (optional: cross-region copy to home region)
   ┌──────────────────────────────────────────────────────────────────┐
   │  Custom training (if MODEL_TYPE=custom_segmentation)             │
   │     - Container: 081189585077.dkr.ecr.us-west-2.amazonaws.com/    │
   │       sagemaker-geospatial:1.0-cpu-py311                          │
   │     - rasterio + geopandas preinstalled                            │
   │     - Train on labeled tiles                                       │
   │     - Output: trained model artifact                                │
   └────────────────┬─────────────────────────────────────────────────┘
                    │
                    ▼ (custom-segmentation case)
   ┌──────────────────────────────────────────────────────────────────┐
   │  Batch Transform on archive                                      │
   │     - Reads scenes from EOJ output bucket                          │
   │     - Runs custom model OR EOJ pre-built model                     │
   │     - Output: classified raster + GeoJSON polygons                 │
   └────────────────┬─────────────────────────────────────────────────┘
                    │
                    ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │  Visualization                                                   │
   │     - Geospatial Studio (us-west-2 only): drag-drop COG viewer    │
   │     - OR custom QGIS / Leaflet using S3 presigned URLs              │
   │     - Tile server (Lambda+API GW) for web embed                     │
   └──────────────────────────────────────────────────────────────────┘
```

---

## Day-by-day execution (5-day POC)

### Day 1 — Region setup + buckets
- Geospatial requires us-west-2 — verify or pivot region
- Output bucket (KMS-encrypted, COG-friendly)
- IAM execution role (Geospatial service principal trust)
- **Deliverable:** sample EOJ submitted for tiny AOI

### Day 2 — EOJ + pre-built model
- EOJ trigger Lambda
- AOI polygon GeoJSON in S3
- For pre-built model use case: configure JobConfig (CloudRemoval / LULC / NDVI / Building / ChangeDetection)
- Run EOJ; verify outputs land in bucket
- **Deliverable:** sample 1° × 1° AOI processed; ~5-15 scenes output

### Day 3 — Custom training (if applicable)
- Custom segmentation training script (rasterio + torch-vision based on architecture choice)
- SageMaker Training Job using `sagemaker-geospatial:1.0-cpu-py311` container
- Tile preparation (256×256 patches with masks)
- Train → register Model Package
- **Deliverable:** trained model artifact + Model Card

### Day 4 — Batch inference + cross-region copy
- Batch Transform on EOJ output bucket (uses custom or pre-built model)
- Output to home region bucket (cross-region S3 replication)
- **Deliverable:** full archive processed; outputs visible in home region

### Day 5 — Visualization + UAT
- Geospatial Studio app (us-west-2 only) for analyst dashboard
- OR custom Lambda + API GW tile server for web visualization
- UAT: domain expert validates classification accuracy on sample AOIs
- **Deliverable:** dashboard with sample classified scenes; UAT signed off

---

## Validation criteria

- [ ] **EOJ throughput** — typical 1° × 1° AOI processes in < 30 min
- [ ] **Output format** — COG (cloud-optimized GeoTIFF) for tile-based serving
- [ ] **Cost cap** — < $50 per AOI processing run
- [ ] **Classification accuracy** (custom training) — > 80% on holdout
- [ ] **Cross-region copy** completes within 6 hr of EOJ output
- [ ] **Visualization** — analyst can pan/zoom + toggle bands within Studio app

---

## Common gotchas (claude must address)

- **us-west-2 only** — Geospatial isn't available in other regions. Pin region.
- **AOI bounds reasonable** — 100° × 100° AOI = thousands of scenes, $$$$. Start small.
- **Cloud cover filter** — without filter, you process clouds. Default 20% max.
- **Output = COG** — don't downgrade to plain GeoTIFF; COG enables web tile serving
- **rasterio version compatibility** — preinstalled in container; don't pip-install another version

---

## Output artifacts

1. CDK app (GeospatialStack)
2. EOJ trigger Lambda
3. AOI GeoJSON sample
4. Custom training script (if applicable) using rasterio + torch
5. Batch Transform job config
6. Cross-region S3 replication config
7. Visualization runbook (Studio app OR custom tile server)
8. Cost comparison: pre-built model ($0.40/scene) vs custom ($X based on training cost)

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-26 | Initial composite. Composes Geospatial ML + Batch Transform + Training. Wave 8. |
