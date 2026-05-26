# 🌊 PyQGIS Llano River Corridor, 150 Structures Terrain Analysis

An automated PyQGIS script that reproduces the full terrain analysis workflow used to identify 150 high-risk structures in the Llano River corridor, Llano and Burnet Counties, TX. The entire QGIS/GRASS workflow, built from scratch using raw federal data sources, runs end to end in a single executable script. This repository is a methodological companion to [ArcPy-Guadalupe-Terrain-Analysis](https://github.com/Austin-AECEomnis/ArcPy-Guadalupe-Terrain-Analysis), applying the same analytical framework to a second Hill Country watershed in a different GIS platform.

---

## 📋 Overview

In October 2018, the Llano River crested more than 10 feet above flood stage, inundating structures throughout the corridor. A terrain analysis conducted in QGIS identified 150 structures sharing elevation, slope, and flow accumulation characteristics consistent with direct inundation risk. The analysis also confirmed a critical data gap: 23 structures in Llano city that were directly inundated fall within FEMA X zones, designating them as moderate-risk rather than high-risk. This mirrors the Hunt VFD finding from Repository 4, where structures in the Guadalupe corridor were similarly misclassified relative to their documented flood exposure.

This script automates that analysis from raw inputs to final output. Given a DEM, FEMA flood zone boundaries, and a building footprint dataset, it derives terrain surfaces, extracts values to structure centroids, and applies the validated filter that produces the 150-structure result.

---

## 🗂️ Repository Contents

| File | Description |
|------|-------------|
| `llano_flood_analysis_pyqgis.py` | Full PyQGIS automation script |
| `README.md` | Methodology, usage, and troubleshooting documentation |

---

## ⚙️ Requirements

| Requirement | Detail |
|-------------|--------|
| Software | QGIS 3.x (tested on 3.34 LTR) |
| Python Environment | QGIS built-in Python console (no separate install required) |
| Plugins | GRASS GIS provider (enabled by default in QGIS) |
| Input 1 | USGS 3DEP 1/3 arc-second DEM as GeoTIFF |
| Input 2 | FEMA NFHL flood zone shapefiles for Llano and Burnet Counties |
| Input 3 | OSM building footprint polygons as GeoJSON |

---

## 📥 Data Sources

All inputs are sourced from freely available federal and open datasets. No proprietary or licensed data is required.

**DEM:** USGS National Map 3DEP 1/3 arc-second elevation data, downloaded via [apps.nationalmap.gov/downloader](https://apps.nationalmap.gov/downloader). Tile covering Llano and Burnet Counties. Stored as `LlanoDEM_raw.tif.tif` (double extension is correct as downloaded).

**Flood Zones:** FEMA National Flood Hazard Layer county shapefiles for Llano County and Burnet County, downloaded from [msc.fema.gov](https://msc.fema.gov). Layer `S_FLD_HAZ_AR.shp` extracted from each county ZIP package.

**Building Footprints:** OpenStreetMap building polygons retrieved via Overpass Turbo ([overpass-turbo.eu](https://overpass-turbo.eu)) using the query `building=*` bounded to the Llano and Burnet County extent. Exported as GeoJSON. OSM coverage in this area is adequate for Llano city and the Kingsland/Lake LBJ corridor but sparse in rural sections of both counties.

---

## 🔁 Workflow Overview

The script executes seven sequential steps:

**Step 1, Reproject flood zone vectors.** Both county flood zone shapefiles are reprojected from their native geographic CRS to EPSG:26914 (NAD83 / UTM Zone 14N). Reprojection to a projected CRS is required before any terrain analysis to ensure distance and area calculations are accurate. The two-county coverage captures the full Llano River corridor from the Kingsland confluence upstream through Llano city.

**Step 2, Convert buildings to centroids and reproject.** OSM building polygons are converted to point centroids, geometry errors are fixed, and the resulting points are reprojected to EPSG:26914. Centroid conversion is necessary because raster sampling tools require point inputs. The geometry fix step handles occasional invalid polygon geometries common in OSM exports.

**Step 3, Reproject DEM.** The raw USGS DEM is reprojected from its native geographic CRS to EPSG:26914 using GDAL Warp with bilinear resampling. Unlike the Guadalupe workflow in Repository 4, no DEM clip step is needed here. The full two-county DEM extent runs within acceptable processing time at this resolution, and clipping to a flood zone boundary would risk excluding building centroids near the zone edges.

**Step 4, Generate Slope.** Slope is derived from the reprojected DEM in degrees using GRASS r.slope.aspect. Values in the study area range approximately 0 to 26.68 degrees. Structures flagged in the final output fall below 20 degrees, consistent with valley floor and low-gradient floodplain positioning. The `ERROR 6 SetColorTable` warning that appears during this step is a harmless GRASS-to-GeoTIFF format artifact and does not affect output accuracy.

**Step 5, Generate Flow Direction and Flow Accumulation.** Both surfaces are derived simultaneously using GRASS r.watershed with a drainage threshold of 10,000 cells. r.watershed uses a Multiple Flow Direction (MFD) algorithm rather than the single-direction D8 method used in ArcGIS Pro. MFD distributes flow across multiple downslope neighbors, which produces more hydrologically realistic accumulation patterns in low-gradient terrain. Negative flow accumulation values in the output are expected MFD behavior and are retained.

**Step 6, Sample terrain values to building centroids.** Elevation, slope, and flow accumulation values are extracted to building centroid points using three sequential Sample Raster Values passes. Each pass adds one field. Three sequential Rename Field passes then standardize column names to `elev_m`, `slope_deg`, and `flow_acc`. This three-pass approach replicates the behavior of ArcGIS Pro's Extract Multi Values to Points tool, which is not available in QGIS.

**Step 7, Apply terrain filter.** Structures are filtered using the expression `elev_m < 420 AND slope_deg < 20 AND flow_acc > 15.88`. The elevation ceiling of 420 meters captures the valley floor extent at both Llano and Kingsland. The slope ceiling of 20 degrees excludes upland and hillside structures not subject to inundation-type flooding. The flow accumulation floor of 15.88 retains structures in zones where upstream drainage converges. These thresholds were calibrated against the documented 2018 inundation extent and validated against the 150-structure count.

---

## ✅ Validated Output

| Metric | Value |
|--------|-------|
| Final structure count | 150 |
| Spatial cluster 1 | Kingsland / Lake LBJ corridor (FEMA A/AE zones) |
| Spatial cluster 2 | Llano city at TX-29/TX-71 (FEMA X zone) |
| Spatial cluster 3 | Burnet city at US-281/TX-29 (FEMA X zone, ambiguous) |
| FEMA zone gap confirmed | 23 structures in Llano city flagged by terrain analysis fall within X zone |
| Target CRS | EPSG:26914 NAD83 / UTM Zone 14N |
| Output format | GeoPackage (.gpkg) |

---

## 🗺️ Key Finding: FEMA X Zone Gap

The most significant output of this analysis is the Llano city cluster. Twenty-three structures at the TX-29/TX-71 confluence fall within FEMA X zones, which designate areas outside the 1% annual chance flood boundary. Despite this designation, the Llano River crested more than 10 feet above flood stage at this location in October 2018, and these structures were directly inundated.

Terrain analysis identified these structures because their elevation, slope, and flow accumulation values place them on the valley floor, in low-gradient terrain, at a drainage convergence point, regardless of what the flood zone boundary says about them. The boundary did not capture their actual exposure.

This finding is methodologically identical to the Hunt VFD result in Repository 4, where 107 structures matching the Camp Mystic terrain signature fell outside or at the edge of AE zone designations despite being inundated in the July 2025 event. The same gap appears in both watersheds, built from independent datasets, using independent tools.

---

## 🔄 Comparison to Repository 4 (ArcPy Guadalupe Terrain Analysis)

| Element | Repository 4 (ArcGIS Pro) | Repository 5 (QGIS) |
|---------|--------------------------|---------------------|
| Platform | ArcGIS Pro 3.x + ArcPy | QGIS 3.x + PyQGIS |
| Flow analysis tool | ArcGIS Flow Direction (D8) + Flow Accumulation | GRASS r.watershed (MFD) |
| Multi-value extraction | ExtractMultiValuesToPoints (single step) | Three sequential Sample Raster Values passes |
| DEM preprocessing | Buffer + clip to study area | Reproject only, no clip needed |
| Output format | File Geodatabase (.gdb) | GeoPackage (.gpkg) |
| Study area | Guadalupe River, Kerr County | Llano River, Llano + Burnet Counties |
| Final structure count | 107 | 150 |
| FEMA zone gap | Hunt VFD corridor, AE zone edge | Llano city, X zone interior |
| Filter logic | Two-pass slope then flow accumulation | Single-expression combining elevation, slope, flow accumulation |

The analytical framework transferred directly between platforms. Tool names, parameter syntax, and output formats differ, but the underlying logic, derive terrain surfaces, sample values to structures, filter by terrain signature, is identical. The primary technical adjustment was replacing r.watershed for the MFD algorithm, which required recalibrating flow accumulation thresholds relative to the D8 values from Repository 4.

---

## ⚠️ Data Limitations

**OSM building coverage.** OpenStreetMap coverage in Llano and Burnet Counties is concentrated in incorporated areas. Rural residential structures along the river corridor between Kingsland and Llano city are substantially underrepresented. The 150-structure count reflects the OSM dataset, not the true number of structures in flood-vulnerable terrain. Higher-quality building footprint data from TNRIS or county appraisal district parcels would produce a more complete result.

**FEMA X zone classification.** The X zone gap identified in this analysis is a finding about the data, not a flaw in the methodology. Terrain analysis surfaces exposure that static flood zone boundaries do not capture. The methodology is sound. More accurate flood zone mapping would produce a tighter alignment between analytical results and official designations, but would not change what the terrain shows.

**Flow accumulation algorithm differences.** GRASS r.watershed MFD values are not directly comparable to ArcGIS D8 flow accumulation values. Thresholds from Repository 4 cannot be applied to Repository 5 outputs without recalibration. Both algorithms correctly identify drainage convergence zones. They measure it differently.

---

## ▶️ How to Run

1. Open QGIS with the Llano project loaded or a new project pointed at your data directory.
2. Open **Plugins > Python Console**.
3. Click the **Editor** button (notepad icon) to open the script editor.
4. Open or paste the contents of `llano_flood_analysis_pyqgis.py`.
5. Confirm the `BASE_DIR` path at the top of the script matches your local folder structure. No other changes are needed if your file names match the expected inputs.
6. Click **Run Script**.

Expected runtime is approximately 5 to 10 minutes. The r.watershed step in Step 5 takes the longest, typically 3 to 4 minutes. A pre-run cleanup step at the top of the script removes any previously loaded layers from the project automatically.

```python
# Only line you may need to change:
BASE_DIR = r"C:\GIS_Projects\LlanoRiver_FloodAnalysis"
```

---

## 🐛 Troubleshooting Notes

**Flow accumulation count mismatch.** The flow_acc column contains the raw MFD accumulation value. On an early script run, a threshold of `flow_acc > 100` produced only 21 structures instead of 150. The validated threshold is `flow_acc > 15.88`. If your count is significantly lower than expected, the flow accumulation threshold is the most likely cause. Print a sample of `flow_acc` values from your Buildings_Final layer before filtering to confirm the value range.

**Double extension on DEM filename.** The USGS National Map download produces a file named with a double `.tif.tif` extension. This is expected. The script references `LlanoDEM_raw.tif.tif` by design. Do not rename the file.

**r.watershed processing time.** GRASS r.watershed on a full two-county DEM runs 3 to 4 minutes with a threshold of 10,000. This is normal. Do not cancel the run. A progress bar is not displayed in the Python console during this step.

**ERROR 6 SetColorTable.** This warning appears after the slope generation step. It is a harmless compatibility note between GRASS raster output and GeoTIFF color table handling. It does not affect the slope values or any downstream steps.

**Null terrain values on output points.** If sampled fields return null for some structures, those centroids fall outside the DEM extent. Confirm the DEM covers the full study area extent before running. The full two-county DEM tile from USGS covers all expected building locations.

---

## 🔗 Related Portfolio Products

This repository is part of a broader multi-platform flood vulnerability analysis series:

- **Repository 4 (ArcGIS Pro):** [ArcPy-Guadalupe-Terrain-Analysis](https://github.com/Austin-AECEomnis/ArcPy-Guadalupe-Terrain-Analysis) — same methodology, ArcGIS Pro platform, Kerr County
- **StoryMap narrative:** https://arcg.is/0bGXv02
- **Experience Builder application:** https://experience.arcgis.com/experience/09c67703781c49ddbc0830655aba9473/
- **Live monitoring dashboard:** https://www.arcgis.com/apps/dashboards/ac7607e8f4fa4185a97697b25cd6b181
- **ArcGIS Online Web Map:** https://aeceomnis.maps.arcgis.com/apps/mapviewer/index.html?webmap=8719efe8dae54c499e312fec5910b899
- **USGS live gauge pipeline:** https://github.com/Austin-AECEomnis/USGS-Guadalupe-LiveStage

---

## 👤 Author

Austin Addington Berlin  
Founder, AECE Omnis LLC  
AI-GIS Convergence Research  
linkedin.com/in/austinberlin  
github.com/Austin-AECEomnis
