---
title: "Solar Panel Analysis"
date: 2025-02-26T18:48:18+01:00
draft: true
---


## Introduction

Have you ever wondered how many households in your neighborhood have a solar panel on their roof and 
how much solar energy is being harnessed?

By combining the power of maps, aerial imagery, solar energy data and machine learning we can address those
questions.

![header](header-319-5653.jpg)

In the image above you see an arial photograph of Titz, North Rhine-Westphalia with 
a almost-transparent overlay highlighting whether a solar panel is installed (green) or not (purple).

## Dataset

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

For further analysis the images taken in the air has to be geo-referenced.
[Georeferencing](https://en.wikipedia.org/wiki/Georeferencing) is the process of assigning real-world coordinates to digital maps or images, 
allowing them to be accurately positioned and overlaid within a geographic information system (GIS).

Luckily, the coordinate information is embedded in every image file. The images are stored in [JPEG2000](https://en.wikipedia.org/wiki/JPEG_2000)
format (`.jp2`) in 10cm x 10cm resolution per pixel. The pixel data (`3xWxH` tensor) and georeferncing information
can be obtained with the [rasterio](https://rasterio.readthedocs.io/en/stable/) Python library.

### Annual solar energy yield

## Methodology
