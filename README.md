# Project: Agricultural Areas in Saudi Arabia (2024)

**Author:** Alaa Labban
**Date:** August 2025
**QGIS Project File:** `final colored.qgz`

---

## 1. Description

This project visualizes agricultural areas across Saudi Arabia for the year 2024. The primary map layer is a raster image representing the Normalized Difference Vegetation Index (NDVI), a key indicator of live green vegetation.

The data was generated using a custom script in Google Earth Engine (GEE). The script processes Landsat 8 satellite imagery, calculates NDVI, and creates a cloud-free composite image for the specified period. This final raster (`NDVI_map_for_WRF.tif`) was then imported into QGIS for final presentation in the `final colored.qgz` project file.

---

## 2. Methodology & Data Generation

The core data for this map was produced in Google Earth Engine. The process is as follows:

1.  **Data Collection**: The script loads the Landsat 8 Collection 2, Tier 1, Level-2 (Surface Reflectance) image collection (`LANDSAT/LC08/C02/T1_L2`).
2.  **Filtering**: It filters the image collection to the geographic bounds of Saudi Arabia and a date range from January 1, 2024, to December 31, 2024.
3.  **Cloud Masking**: A function (`maskL8sr`) is applied to each image to remove pixels obscured by clouds and cloud shadows, using the `QA_PIXEL` band.
4.  **NDVI Calculation**: For each clean image, NDVI is calculated using the formula `(NIR - Red) / (NIR + Red)`. This corresponds to Band 5 (NIR) and Band 4 (Red) in Landsat 8 imagery.
5.  **Composite Image**: A single, clean composite image is created by taking the median NDVI value for every pixel across the entire year-long collection. This method helps to create a clear view that represents the typical vegetation state.
6.  **Export**: The final NDVI raster is exported as a GeoTIFF file with a 30-meter resolution.

---

## 3. Software & Platforms

* **Google Earth Engine:** Used for all data processing and generation of the NDVI raster.
* **QGIS:** Used for visualizing, styling, and presenting the final map in the `.qgz` project file.

---

## 4. How to View the Map

1.  Ensure you have QGIS installed on your system.
2.  Navigate to the project folder and double-click the `final colored.qgz` file.
3.  QGIS will launch and load the map with all layers and styling preserved.

---

## 5. Map Layer

* **Layer: `NDVI_map_for_WRF`**
    * **Description:** This is the primary raster layer showing median NDVI values for 2024. Higher values (greens) indicate denser vegetation, which in the context of Saudi Arabia's arid environment, strongly correlates with agricultural activity. Lower values (browns) represent barren land, desert, or urban areas.
    * **Data Source:** Landsat 8 Surface Reflectance imagery, processed in Google Earth Engine.
    * **Resolution:** 30 meters.

---

## 6. Google Earth Engine Script

The following script was used to generate the data. It can be run in the [GEE Code Editor](https://code.earthengine.google.com/) by defining a geometry for Saudi Arabia.

<details>
<summary>Click to view the full GEE script</summary>

```javascript
// =================================================================
//  Google Earth Engine Script to Identify and Export Agricultural Areas
//  (Final version with maxPixels enabled for large export)
// =================================================================

// --- Part 1: Function to Mask Clouds ---
function maskL8sr(image) {
  var cloudShadowBitMask = (1 << 3);
  var cloudsBitMask = (1 << 5);
  var qa = image.select('QA_PIXEL');
  var mask = qa.bitwiseAnd(cloudShadowBitMask).eq(0)
                 .and(qa.bitwiseAnd(cloudsBitMask).eq(0));
  return image.updateMask(mask);
}

// --- Part 2: Function to Calculate NDVI ---
var calculateNDVI = function(image) {
  var nir = image.select('SR_B5');
  var red = image.select('SR_B4');
  var ndvi = nir.subtract(red).divide(nir.add(red)).rename('NDVI');
  return image.addBands(ndvi);
};

// --- Part 3: Load Data and Create a Composite Image ---
// Note: 'geometry' must be defined in the GEE editor before running.
var collection = ee.ImageCollection('LANDSAT/LC08/C02/T1_L2')
    .filterBounds(geometry)
    .filterDate('2024-01-01', '2024-12-31')
    .map(maskL8sr)
    .map(calculateNDVI);

var ndviComposite = collection.select('NDVI').median();
var clippedNdvi = ndviComposite.clip(geometry);

// --- Part 4: Visualize the Result ---
var ndviPalette = [
  '#a52a2a', // Brown (low NDVI)
  '#ffff00', // Yellow
  '#00ff00', // Green
  '#006400'  // Dark Green (high NDVI)
];

Map.addLayer(clippedNdvi, {min: 0, max: 0.6, palette: ndviPalette}, 'NDVI Map (Agricultural Areas)');
Map.centerObject(geometry, 8);

// --- Part 5: Export the Final Image to Your Google Drive ---
Export.image.toDrive({
  image: clippedNdvi,
  description: 'NDVI_map_for_WRF',
  folder: 'GEE_Exports',
  scale: 30,
  region: geometry,
  fileFormat: 'GeoTIFF',
  maxPixels: 10000000000 // <-- Allows for a very large export
});

</details>
