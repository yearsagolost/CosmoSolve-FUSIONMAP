import geemap
import ee

ee.Authenticate()
ee.Initialize(project='hackathon-480419')

baku_roi = ee.Geometry.Polygon([
    [[49.65, 39.85],
     [50.35, 39.85],
     [50.35, 40.6],
     [49.65, 40.6],
     [49.65, 39.85]]
])

# Sentinel-2 RGB
sentinel2 = ee.ImageCollection('COPERNICUS/S2_SR') \
    .filterDate('2025-01-01', '2025-12-12') \
    .filterBounds(baku_roi) \
    .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 50)) \
    .select(['B4','B3','B2'])

optical_image = sentinel2.median().clip(baku_roi)

# Normalize to 0-1 for proper visualization
optical_norm = optical_image.divide(3000).clamp(0,1)

# MODIS Thermal
modis = ee.ImageCollection('MODIS/006/MOD11A1') \
    .filterDate('2025-01-01', '2025-12-12') \
    .filterBounds(baku_roi) \
    .select('LST_Day_1km') \
    .map(lambda img: img.multiply(0.02).subtract(273.15))

optical_vis = {'bands':['B4','B3','B2']}
thermal_vis = {'min':20, 'max':60, 'palette':['blue','yellow','red']}

Map = geemap.Map(center=[40.4, 50.0], zoom=10)
Map.addLayer(optical_norm, optical_vis, 'Optical', True, 0.8)

if modis.size().getInfo() > 0:
    modis_mean = modis.mean().clip(baku_roi)
    fire_mask = modis_mean.gt(60)
    fire_exists = fire_mask.reduceRegion(
        reducer=ee.Reducer.anyNonZero(),
        geometry=baku_roi,
        scale=1000
    ).getInfo()['LST_Day_1km']
    if fire_exists:
        Map.addLayer(fire_mask.updateMask(fire_mask), {'palette':['red']}, 'Fire Hotspots', True, 0.6)
        Map.addLayer(modis_mean, thermal_vis, 'Thermal (°C)', True, 0.5)

Map.addLayerControl()
Map
