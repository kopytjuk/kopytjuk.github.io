---
title: "Solar Panel Analysis"
date: 2025-02-26T18:48:18+01:00
draft: true
math: true
---


## Introduction

Have you ever wondered how many households in your neighborhood have a solar panel on their roof and 
how much solar energy is being harnessed?

By combining the power of maps, aerial imagery, solar energy data and machine learning we can address those
questions.

![header](header-319-5653.jpg)

In the image above you see an arial photograph of Titz, North Rhine-Westphalia with 
a almost-transparent overlay highlighting whether a solar panel is installed (green) or not (purple).

## Datasets

The data for this analysis is downloaded from the [OpenGeodata.NRW](https://www.opengeodata.nrw.de/produkte/) portal. 

The OpenGeoData NRW portal is maintained by the state government of North Rhine-Westphalia (NRW) in Germany. 
It serves as a platform for providing open geospatial data to the public, 
aligning with the principles of Open Government, which emphasize transparency and public participation.

The portal offers a wide range of geospatial data, including high-resolution aerial images,
LIDAR measurements, land-property and solar yield maps.
Based on LIDAR data, also derived datasets such as digital surface and terrain models are provided.

However, in this blog post the focus of this analysis will be on aerial images and solar yield maps.
The next two sections will provide a basic introduction into both data types.

### Aerial images

Aerial photography (or airborne imagery) is the taking of photographs from an aircraft or other airborne platforms such as drones.
An aerial image is usually converted to an [**orthophotograph**](https://en.wikipedia.org/wiki/Orthophoto) in a process called
**orthorectification**. Unlike an uncorrected aerial photograph, an orthophoto can be used to measure true distances. In other words,
when we measure the distance between two trees, we can multiple the value in pixel with a known scaling factor to obtain the true distance in meters.

For simplicity, I will use the term *aerial image* to refer to both aerial photographs and orthophotographs in this article.

The image below shows an orthophoto of a residential area close to the margin of a a single orthophoto. 
Note that the buildings are not perfect rectangles, due to the orthorectification process.

![orthophoto](orthophoto-example.jpg)

For further analysis (e.g. to find a building to a particular location) the images have to be geo-referenced.
[Georeferencing](https://en.wikipedia.org/wiki/Georeferencing) is the process of 
assigning real-world coordinates to digital maps or images, 
allowing them to be accurately positioned and overlaid within a geographic information system (GIS).

Luckily, the coordinate information is already embedded in every image file allowing us to locate a particular location (e.g. a GPS longitude and latitude) to a single pixel.
The image data and georeferencing metadata are stored in [JPEG2000](https://en.wikipedia.org/wiki/JPEG_2000) format (`.jp2`).

You can access the images [here](https://www.opengeodata.nrw.de/produkte/geobasis/lusat/akt/dop/dop_jp2_f10/).

### Annual solar energy yield

In order to estimate the amount of energy a roof covered by solar panels generates, we will
use the solar energy yield (in German: *Solarkataster*) data. Each pixel of this bitmap contains the expected annual energy yield in $\frac{kWh}{m^2}$. The values were estimated using solar radiation and weather data provided by the German Weather Service combined with
digital surface models for considering obstacles like trees or higher buildings which reduce the solar yield by casting shadows. More details on how those maps are created can be found in [1].

The image below shows aerial image data overlayed with energy yield data. Dark red areas indicate high energy yield where white areas
indicate low solar returns.

![solar-yield](solar-yield-example.jpg)

The bitmaps are available in 50cm and 1m resolution (true length of each pixel). You can access the yield data [here](https://www.opengeodata.nrw.de/produkte/umwelt_klima/energie/solarkataster/strahlungsenergie_50cm/).

### Data organization

The area of state North Rhine-Westphalia is ca. 34000km. Since storing all the data in a single file is not
really realistic (think of a single aerial image with 10cm resolution per pixel), thus both the images and bitmaps
are organized in **tiles**. A tile covers a square with 1km or 4km side length.

The following image shows the 1km-tile coverage of aerial images of the city of Cologne:

![tiles-cologne](tiles-cologne.jpg)

The tile information is encoded in the file name, e.g. here for a single 
aerial image `dop10rgbi_32_280_5652_1_nw_2023.jp2`:

- `dop`: digital orthophotograph (i.e. aerial image)
- `10`: 10cm resolution
- `rgbi`: RGB with an infrared channel
- `32`: UTM32 (EPSG 25832) projection
- `280`: x-coordinate south west corner of the tile in km, here 280000m
- `5652`: y-coordinate south west corner of the tile in km, here 5652000m
- `1`: side length of the tile, here 1km
- `nw`: tile orientation, here north-west
- `2023`: recording year

## Methodology

The following workflow outlines the processing of a single 1km tile.

![methodology](methodology.png)

First, all buildings, address, their exact location and their polygon outlines (e.g. a rectangle for a simple 4 wall building) are extracted from OpenStreetMap (OSM).
Each building receives a unique ID. The outline is used for cropping images and for extracting the solar yield.

The area around a single building is cropped from the aerial image tiles. The  images are passed to an Machine Learning (ML) based object detector, which outputs the location of each solar panel group.

The detections are combined with solar energy yield data (and several assumptions about the solar panel technology) to estimate the **actual** annual energy yield. The building outlines combined with solar energy yield data provide the **potential** amout of energy.

The information flows into a final tabular dataset (table in grey) which can be further employed to answer
our introductory questions about installation rates of solar panels.

The subsequent sections will offer technical insights into each step. If you are interested in the technical modalities like masking raster data or selecting the right ML model, feel free to stay. Otherwise, feel free to skip ahead to the [Case Study: X](#case-study-x) section,
where we use the final table to analyze the installation rate of a small village.

### Extracting building outlines

First, we need to extract all buildings which exist in a single geographical area. For that, we will use
the OSMnx library, a Python package to access street networks and other geospatial features (such as buildings in our case) 
from OSM:

```python
import osmnx as ox

# Fetch buildings from OpenStreetMap
buildings_gdf = ox.features_from_bbox(bbox, tags={"building": True})
```

The result is a [GeoDataFrame](https://geopandas.org/en/stable/docs/reference/api/geopandas.GeoDataFrame.html) 
with the OSM `way` identifier as its index. We keep the original identifier of the building, in order to find it in OpenStreetMap, e.g.
for the building `316174397` we can find it at https://www.openstreetmap.org/way/316174397.

The `geometry` column of the `buildings_gdf` holds the building outline in WGS84 geodetic coordinates,
i.e. it is a polygon represented as a sequence of (longitude, latitude) pairs. Note that the units of WGS84 coordinates
are degrees and not lengths such as meters or feet. 

In order to use the building geometries with the geographic data from OpenGeodata.NRW, they first need to be transformed
to the UTM32N (Northern Hemisphere) coordinate system.
The [UTM](https://en.wikipedia.org/wiki/Universal_Transverse_Mercator_coordinate_system)-system (UTM32N is a subset) is a cartesian system, which can be used to measure lengths in meters.

For the transformation we implement two helpful functions which we will use across the project:

```python
import pyproj
from shapely import ops as shapely_ops
from shapely.geometry.base import BaseGeometry

# WGS84 to UTM
transformer_to_25832 = pyproj.Transformer.from_crs("EPSG:4326", 
    "EPSG:25832", always_xy=True)

def transform_wgs84_to_utm32N(geom: BaseGeometry) -> BaseGeometry:
    return shapely_ops.transform(transformer_to_25832.transform, geom)
```

The `transform_wgs84_to_utm32N` function can transform any of [shapely](https://shapely.readthedocs.io/en/2.0.6/index.html)'s geometry types (Point, LineString, Polygon) from WGS84 (i.e. longitude and latitude) to the desired UTM32 (x, y) coordinates. Using this function applied on the geometries in `buildings_gdf` we can compile a a tabular **building overview** dataset, which consists of a set of buildings with following attributes:

- building ID
- OSM way ID
- address (`addr:street`, `addr:housenumber`, `addr:postcode` etc.)
- building outline in UTM32 coordinates (as a `shapely.Polygon`)

The following image shows the first two entries of the building overview data:

![building-overview](building-overview-df.jpg)

The dataset is used as an input for both the cropping image and energy extraction steps.

### Cropping aerial images

The ML-model which I selected for the detection tasks works well for a zoomed in image of a building. Therefore, I need to crop the large aerial image from a single tile into smaller images, having the building of interest in its center with a 5-10m margin around.

To open the aerial image we will use the [rasterio](https://rasterio.readthedocs.io/en/stable/) Python library.
The library interprets both the pixel data as georeferencing information, which is is stored as metadata in the JPEG2000 file.
The pixel data can be accessed with a `.read()` call in a `4xWxH` tensor, with red, blue, green and infrared channels.
`W` and `H` are the width and height of the image in pixel units.

The following script shows the steps required to crop a single building from the:

```python
import rasterio


building_outline: Polygon # from the previous step
output_location = Path("./cropped-images")

aerial_image_path = "dop10rgbi_32_280_5652_1_nw_2023.jp2"

with rasterio.open(aerial_image_path) as image_data:

    # transform pixel coordinates to UTM32 coordinates (georeference metadata)
    affine_transform_px_to_geo = image_data.transform

    building_outline_with_margin = building_outline.envelope.buffer(5.0)

    # pixel area to cut from (holds the bounding box of the cut)
    crop_window = rasterio.windows.from_bounds(
                    *building_outline_with_margin.bounds,
                    transform=affine_transform_px_to_geo,
                )

    # read the image
    image_matrix = image_data.read(window=crop_window)

    # ...

    # store the image
    plt.imsave("building-cut.png", arr=image_matrix, dpi=200)
```

Additionally to the cropped image, using `affine_transform_px_to_geo` object we transform 
the building outline to pixel coordinates of the cropped image. The transformed polygon is later used
to filter out detections which fall outside the area of the actual building:

```python
# invert
affine_transform_geo_to_px = ~affine_transform_px_to_geo

# convert to a structure used by shapely
affine_transform_for_shapely = affine_transform_geo_to_px.to_shapely()

# polygon in pixel coordinates of the full tile
building_polygon_px = shapely.affinity.affine_transform(
    building_polygon, affine_transform_for_shapely
)

# polygon in the pixel coordinates of the cropped image
building_polygon_px_image = shapely.affinity.translate(
    building_polygon_px, -crop_window.col_off, -crop_window.row_off
)
```

The result of the cropping logic is a folder with images (with building-IDs as filenames) and a
`overview.csv` table which holds the building polygon coordinates in pixel coordinates:

![](image-cropper-output.jpg)

This folder is used as input for the ML-based detector, which we discuss next.

### Running an ML detector

First 

### Retrieving solar yield

Similar to the aerial images, the solar yield bitmaps can be processed in a similar manner with [rasterio](https://rasterio.readthedocs.io/en/stable/). The bitmap data with width W and height H is stored in a `WxH` matrix.

### Join all information


## Case study: X


## Summary


## Lessons learned and possible improvements


- group detect for multiple buildings at once
- use segmentation models
- Alternatively, at the detection step
we could transform the detection box back to UTM32. Store Afiinity Matrix Instead (smaller)

## Similar projects

- https://www.appsilon.com/post/using-ai-to-detect-solar-panels-part-1 (uses segmentation)

## References

[1] [LANUV Info 43](https://www.energieatlas.nrw.de/site/service/download_publikationen) (PDF)
