# Spotting Crop Stress from Space: Vegetation Indices with Sentinel-2 and Earth Engine

Crop stress often shows up in satellite data before a farmer can see it from the edge of the field. In this tutorial, you'll compute three vegetation indices for a single field using free Sentinel-2 imagery and the Google Earth Engine Python API, then compare the result against the same field's own history.

This is the approach behind the satellite layer of **Oh Deere!**, the precision agriculture app my team built in 72 hours at DonsHack '26. The code below is adapted from our backend (`backend/services/earth_engine.py`).

**What you'll build:** a Python function that takes a field boundary and a date range and returns the average NDVI, NDRE, and NDMI for that field, plus a second step that tells you whether this month is unusual for that field.

**Time:** about 20 minutes, most of it one-time Earth Engine setup.

---

## Prerequisites

- Python 3.10+
- A Google Cloud project with the Earth Engine API enabled (free for non-commercial and research use)
- The Earth Engine client library:

```bash
pip install earthengine-api
earthengine authenticate
```

---

## Background: what the three indices measure

Healthy leaves reflect a lot of near-infrared (NIR) light and absorb red light for photosynthesis. Each index compares two Sentinel-2 bands to pull out a different signal:

| Index | Bands (Sentinel-2) | What it tells you |
|-------|--------------------|-------------------|
| **NDVI** | (NIR − Red) / (NIR + Red) → B8, B4 | Overall vegetation density and vigor |
| **NDRE** | (NIR − Red Edge) / (NIR + Red Edge) → B8, B5 | Chlorophyll content; often reacts earlier than NDVI in dense crops |
| **NDMI** | (NIR − SWIR) / (NIR + SWIR) → B8, B11 | Water content in the canopy; useful for spotting drought stress |

All three range from −1 to 1. For crops, higher generally means healthier, denser, or wetter vegetation. No single index tells the whole story, which is why we compute all three.

---

## Step 1: Connect to Earth Engine and define your field

```python
import ee

ee.Initialize(project="your-gcp-project-id")

# Earth Engine expects [longitude, latitude] pairs, and the ring must close
field = ee.Geometry.Polygon([[
    [-121.75, 38.55],
    [-121.74, 38.55],
    [-121.74, 38.54],
    [-121.75, 38.54],
    [-121.75, 38.55],
]])
```

> **Gotcha we hit at the hackathon:** most web map libraries (we used Leaflet) give you coordinates as `[lat, lon]`, but Earth Engine wants `[lon, lat]`. If your field shows up in the ocean, swap them.

---

## Step 2: Load cloud-free Sentinel-2 imagery

```python
images = (
    ee.ImageCollection("COPERNICUS/S2_SR_HARMONIZED")
    .filterBounds(field)
    .filterDate("2026-06-01", "2026-06-30")
    .filter(ee.Filter.lt("CLOUDY_PIXEL_PERCENTAGE", 20))
)

print("Images found:", images.size().getInfo())
```

Sentinel-2 revisits each location about every five days, so a one-month window usually gives you several images. If the count is 0, widen the date range or loosen the cloud filter.

---

## Step 3: Compute the indices

Earth Engine's `normalizedDifference` does the `(a − b) / (a + b)` math for you. We add all three indices to every image, then take the median across the month, which also helps filter out leftover clouds and shadows.

```python
def add_indices(image):
    ndvi = image.normalizedDifference(["B8", "B4"]).rename("NDVI")
    ndre = image.normalizedDifference(["B8", "B5"]).rename("NDRE")
    ndmi = image.normalizedDifference(["B8", "B11"]).rename("NDMI")
    return image.addBands([ndvi, ndre, ndmi])

composite = images.map(add_indices).median()

stats = composite.select(["NDVI", "NDRE", "NDMI"]).reduceRegion(
    reducer=ee.Reducer.mean(),
    geometry=field,
    scale=10,          # meters per pixel
    maxPixels=1e9,
).getInfo()

print(stats)
# {'NDMI': 0.21, 'NDRE': 0.38, 'NDVI': 0.71}   (example output)
```

> **Note on resolution:** Sentinel-2's red and NIR bands are captured at 10 m, but the red-edge (B5) and SWIR (B11) bands are 20 m. Earth Engine resamples them when you request `scale=10`, so treat NDRE and NDMI as slightly coarser than NDVI.

---

## Step 4: Compare against the field's own history

A single NDVI value of 0.55 doesn't mean much on its own: it could be excellent for one crop in May and a warning sign for another in July. What matters is whether it's unusual **for this field at this time of year**.

Oh Deere! pulls three years of monthly NDVI averages for each field. Here's a simplified version that compares this June to the previous three Junes:

```python
def monthly_ndvi(year, month):
    start = ee.Date.fromYMD(year, month, 1)
    month_images = (
        ee.ImageCollection("COPERNICUS/S2_SR_HARMONIZED")
        .filterBounds(field)
        .filterDate(start, start.advance(1, "month"))
        .filter(ee.Filter.lt("CLOUDY_PIXEL_PERCENTAGE", 20))
        .map(lambda img: img.normalizedDifference(["B8", "B4"]).rename("NDVI"))
    )
    value = month_images.mean().reduceRegion(
        ee.Reducer.mean(), field, scale=10, maxPixels=1e8
    ).get("NDVI")
    return value.getInfo()

current = monthly_ndvi(2026, 6)
history = [monthly_ndvi(y, 6) for y in (2023, 2024, 2025)]
baseline = sum(history) / len(history)

change = (current - baseline) / baseline * 100
print(f"June NDVI: {current:.2f} vs. 3-year June average {baseline:.2f} ({change:+.1f}%)")
```

If this June's NDVI is well below the three-year June average, that part of the season deserves a closer look, even if the field still looks green from the road.

---

## Interpreting what you see

- **NDVI down, NDMI down:** the canopy is thinning and drying out. Check recent rainfall and irrigation first.
- **NDRE down, NDVI steady:** chlorophyll may be dropping before the canopy visibly changes. Nutrient stress (especially nitrogen) is a common cause.
- **All three down in one corner only:** look for a local cause like poor drainage, compaction, or pests, and scout that spot in person.

Satellite indices tell you **where** to look, not **what** is wrong. They're a starting point for a field visit, not a replacement for one.

---

## Where to go next

- **Add weather context.** We paired the indices with 14-day forecasts from the free [Open-Meteo API](https://open-meteo.com/) so low NDMI could be read alongside actual rainfall.
- **Add soil context.** [SoilGrids](https://soilgrids.org/) provides soil texture, pH, and organic carbon for any point on Earth, which helps explain why one part of a field struggles more than another.
- **Make it visual.** Instead of one average per field, export the composite as a map layer so stress zones are visible at a glance.

The full Oh Deere! source code is at [github.com/sucue4/donshack26](https://github.com/sucue4/donshack26).
