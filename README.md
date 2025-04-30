# 🌍 **Land Cover Classification and Change Detection using Landsat 5 and Landsat 8** 🌍

This project uses **Landsat 5** (2000) and **Landsat 8** (2023) imagery to perform **land cover classification** and **change detection** over time. The goal is to compare and analyze changes in land cover between two different time periods, identifying different classes such as **Built-up**, **Bareland**, **Water**, **Parks**, **Agriculture**, and **Mountains**.

By leveraging satellite imagery, **spectral indices** like **EVI**, **NBR**, **NDWI**, **NDBI**, and others, this project allows you to analyze urbanization, vegetation, and other land cover types. We also use **Random Forest Classification** to identify and predict land use classes and calculate **land cover area statistics** for both 2000 and 2023.

---

## 🚀 **Key Features**

- **Cloud Masking**: Pre-process satellite images by masking clouds and other atmospheric effects.
- **Land Cover Classification**: Classify the land cover using **Random Forest Classifier** based on various spectral indices.
- **Change Detection**: Compare land cover classes between **2000** and **2023** to detect changes.
- **Export Results**: Export the classified images and sample data as **GeoTIFF** and **CSV** files.
- **Spectral Indices**: Calculate various indices like **EVI**, **NBR**, **NDWI**, and **NDBI** to better understand vegetation, built-up areas, and water bodies.

---

## 🧑‍🔬 **Getting Started**

### 1. **Sign up for Google Earth Engine (GEE)**

If you don't already have a **Google Earth Engine (GEE)** account, sign up [here](https://signup.earthengine.google.com/).

### 2. **Run the Script**

1. **Open the [Google Earth Engine Code Editor](https://code.earthengine.google.com/)**.
2. Paste the provided code into the editor.
3. **Run** the script to begin the analysis, which will process **Landsat 5** (2000) and **Landsat 8** (2023) data to generate classified land cover maps and calculate land cover area statistics.

---

## 🔧 **How It Works**

### 1. **Cloud Masking**

The script applies a **cloud masking function** to remove cloud cover and other atmospheric interference from the images. The mask uses the **QA_PIXEL** band from Landsat data to flag dilated clouds, cirrus, cloud shadows, and other unwanted features.

```javascript
function cloudMask(image) {
  var qa = image.select('QA_PIXEL');
  var dilated = 1 << 1;
  var cirrus = 1 << 2;
  var cloud = 1 << 3;
  var shadow = 1 << 4;
  var mask = qa.bitwiseAnd(dilated).eq(0)
    .and(qa.bitwiseAnd(cirrus).eq(0))
    .and(qa.bitwiseAnd(cloud).eq(0))
    .and(qa.bitwiseAnd(shadow).eq(0));
  return image.select(['SR_B.*'], ['B1', 'B2', 'B3', 'B4', 'B5', 'B6', 'B7'])
    .updateMask(mask)
    .multiply(0.0000275)
    .add(-0.2);
}
```

### 2. **Image Collection and Cloud Masking**

Landsat data for **2000** (Landsat 5) and **2023** (Landsat 8) is loaded, filtered by **date** and **region**, and cloud-masked using the `cloudMask` function:

```javascript
var image = l8.filterBounds(roi).filterDate('2023-01-01', '2023-12-31')
  .merge(l9.filterBounds(roi).filterDate('2023-01-01', '2023-12-31'))
  .map(cloudMask)
  .median()
  .clip(roi);
```

### 3. **Spectral Indices Calculation**

The script calculates multiple **spectral indices** including **EVI**, **NBR**, **NDMI**, **NDWI**, **NDBI**, and **NDBaI** using formulas based on different bands (such as **NIR**, **SWIR**, **RED**, etc.). These indices help highlight areas like urban growth, water bodies, and vegetation:

```javascript
var indices = ee.Image([
  { name: 'EVI', formula: '(2.5 * (NIR - RED)) / (NIR + 6 * RED - 7.5 * BLUE + 1)' },
  { name: 'NBR', formula: '(NIR - SWIR2) / (NIR + SWIR2)' },
  { name: 'NDMI', formula: '(NIR - SWIR1) / (NIR + SWIR1)' },
  { name: 'NDWI', formula: '(GREEN - NIR) / (GREEN + NIR)' },
  { name: 'NDBI', formula: '(SWIR1 - NIR) / (SWIR1 + NIR)' },
  { name: 'NDBaI', formula: '(SWIR1 - SWIR2) / (SWIR1 + SWIR2)' },
].map(function(dict) {
  var indexImage = image.expression(dict.formula, bandMap).rename(dict.name);
  return indexImage;
}));
```

### 4. **Training and Testing Data for Classification**

The **training samples** are derived from different land cover types (built-up, bareland, water, parks, agriculture, mountains) and split into **training** and **test** sets. The **Random Forest** model is trained with the selected features and then used to classify the entire image.

```javascript
var classifier = ee.Classifier.smileRandomForest(300).train({
  features: trainSet,
  classProperty: 'classvalue',
  inputProperties: bands
});
```

### 5. **Land Cover Classification**

The script classifies the image using the **Random Forest** classifier and displays the classified land cover image on the map. The result includes **Built-up**, **Bareland**, **Water**, **Parks**, **Agriculture**, and **Mountains** as the class labels:

```javascript
var classified = image.classify(classifier);
Map.addLayer(classified, {
  min: 1,
  max: 6,
  palette: ['#F08080', '#D2B48C', '#008080', '#90EE90', '#FF0000']
}, 'Classified Image');
```

### 6. **Change Detection and Area Calculation**

The script performs **change detection** by comparing land cover classes between **2000** and **2023** and calculates the area for each land cover type (e.g., built-up areas, agricultural areas):

```javascript
var lulcBuiltUpArea = lc.eq(1).multiply(ee.Image.pixelArea()).reduceRegion({
  reducer: ee.Reducer.sum(),
  geometry: roi,
  scale: 30,
  bestEffort: true
}).get('lulc');
```
![land use 2000](https://github.com/user-attachments/assets/cc1c8cef-d491-4a07-a6be-1c155c706b4a)

![land use 2023](https://github.com/user-attachments/assets/80422bcd-c384-4157-a738-f2aa439778b2)


Charts comparing **built-up areas**, **agriculture**, **water**, and other land cover types between **2000** and **2023** are generated.

### 7. **Export Results**

Finally, the classified images and sample data are exported as **GeoTIFF** and **CSV** files to **Google Drive** for further analysis:

```javascript
Export.image.toDrive({
  image: lc.toInt32(),
  description: 'LULC_2000',
  folder: 'GeoTIFF',
  region: roi,
  scale: 30,
  crs: 'EPSG:4326',
  maxPixels: 1e13
});
```

---

## 📊 **Outputs**

### 1. **Classified Land Cover Image (2000 and 2023)**

A classified image representing different land cover types (Built-up, Bareland, Water, etc.) for both **2000** and **2023**.

### 2. **Land Cover Area Statistics**

The total area (in square meters) for each land cover class in **2000** and **2023**.

### 3. **Change Detection Charts**

Charts comparing the area of different land cover classes (e.g., built-up, agriculture) between **2000** and **2023**.

---

## 📈 **Future Enhancements**

- **Time-Series Analysis**: Extend the analysis to cover multiple years and assess long-term land cover changes.
- **Machine Learning Models**: Implement more advanced machine learning models for improved accuracy in land cover classification.
- **High-Resolution Data**: Integrate **Sentinel-2** or higher resolution imagery for more detailed analysis.

---

