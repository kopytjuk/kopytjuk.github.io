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

The image below shows an orthophoto of a residential area close to the margin of a a single orthophoto. 
Note that the buildings are not perfect rectangles, due to the orthorectification process.

![orthophoto](orthophoto-example.jpg)

For further analysis (e.g. to find a building to a particular location) the images have to be geo-referenced.
[Georeferencing](https://en.wikipedia.org/wiki/Georeferencing) is the process of 
assigning real-world coordinates to digital maps or images, 
allowing them to be accurately positioned and overlaid within a geographic information system (GIS).

Luckily, the coordinate information is already embedded in every image file allowing us to locate a particular location (e.g. a GPS longitude and latitude) to a single pixel.
The image data and georeferencing metadata are stored in [JPEG2000](https://en.wikipedia.org/wiki/JPEG_2000) format (`.jp2`).

You can find the areal images [here](https://www.opengeodata.nrw.de/produkte/geobasis/lusat/akt/dop/dop_jp2_f10/).

### Annual solar energy yield

In order to estimate the amount of energy a roof covered by solar panels generates, we will
use the solar energy yield (in German: *Solarkataster*) data. Each pixel of this bitmap contains the expected annual energy yield in $\frac{kWh}{m^2}$. The values were estimated using solar radiation and weather data provided by the German Weather Service combined with
digital surface models for considering obstacles like trees or higher buildings which reduce the solar yield by casting shadows. More details on how those maps are created can be found in [1].

The image below shows orthophoto data overlayed with energy yield data. Dark red areas indicate high energy yield where white areas
indicate low solar returns.

![solar-yield](solar-yield-example.jpg)

The bitmaps are available in 50cm and 1m resolution (true length of each pixel). You can find the yield data [here](https://www.opengeodata.nrw.de/produkte/umwelt_klima/energie/solarkataster/strahlungsenergie_50cm/).

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

The following image shows the 

```python
print("hello")
```

### Finding building outlines

### Cropping the images

The pixel data and georeferecing information
can be obtained with the [rasterio](https://rasterio.readthedocs.io/en/stable/) Python library.

 The pixel data
of an image with width W and height H is stored in a `4xWxH` tensor, with red, blue, green, infrared channels.

### Running an ML detector

### Retrieving solar yield

Similar to the aerial images, the solar yield bitmaps can be processed in a similar manner with [rasterio](https://rasterio.readthedocs.io/en/stable/). The bitmap data with width W and height H is stored in a `WxH` matrix.

### Join all information

### Overview


## References

[1] [LANUV Info 43](https://www.energieatlas.nrw.de/site/service/download_publikationen) (PDF)
