---
layout: post
title: Bathymetric AR App
date: 2021-09-13T17:21:03+0300
description: This blog post introduces my side project called bathymetric-cam. bathymetric-cam is an iOS AR app visualizing depth contour of water.
tags: iOS app fishing
categories: MobileApp
giscus_comments: true
related_posts: false
toc:
  sidebar: left
---

This blog post introduces my side project called bathymetric-cam. bathymetric-cam is an iOS AR app visualizing depth contour of water.

## Motivation

Have you ever wanted to know the water depth on the site?

It may sound a bit weird to you but yes, I have. Especially for anglers, depth contour can be a hint to find the target species. Depth contour represents a line connecting points of equal depth on ocean or lake floors. If the contour lines spaced narrowly between one and another, it indicates that the area has the steep transition. This kind of spots is called drop-offs when the anglers target a certain type of species like Largemouth Bass. Also, for example, Largemouth Bass is known as the habit preferring to stay at deeper water when the temperature is cold in winter. bathymetric-cam visualizes the depth contour by the intuitive AR view in order to help you find a better fishing spot.

## Prototyping

### Data Creation

To visualize the water depth, first, I need the data for POC. I drew the depth contour polygon by hand on [QGIS](https://www2.qgis.org) app, and exported GeoJSON file per map tile based on the [tile system](https://wiki.openstreetmap.org/wiki/Slippy_map_tilenames).

```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "properties": { "maxDepth": 2.0, "minDepth": 1.5 },
      "geometry": { "type": "MultiPolygon", "coordinates": ... }
    },
    ...
  ]
}
```

Then I wrote a [script](https://github.com/bathymetric-cam/geojson-to-map-tile) converting GeoJSON files into PNG map tile images. For example, the following GeoJSON file becomes the map tile image below.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="/assets/img/2021-09-13-bathymeric-ar-app-geojson.jpg" title="GeoJSON" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="/assets/img/2021-09-13-bathymeric-ar-app-maptile.png" title="Map tile" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

As a result, I have drawn a part of the south lake in [Lake Biwa](https://en.wikipedia.org/wiki/Lake_Biwa). The manual labor is definitely not for a lazy programmer…

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="/assets/img/2021-09-13-bathymeric-ar-app-qgis.webp" title="Map tiles" class="img-fluid" %}
    </div>
</div>

The PNG files are uploaded on CDN. It works as a simplified map tile server.

### iOS app

The prototype app is a simple AR app that has a rounded map on the bottom. If you turn your iPhone’s camera toward one direction, the map follows the exact same direction. Thus the AR and map seamlessly face to the same direction. They also render the same map tiles downloaded from the tile server. I place a slider UI that adjusts the altitude of water surface. Let’s assume the anchor point is where your camera is. The water surface is located under X meter of it.

{% include video.html path="https://www.youtube.com/embed/HrZpjp9iqkA" class="img-fluid rounded z-depth-1" %}

## Foresight

Honestly, I’m struggling to decide where to make the next improvement. The jaggy image isn’t looking very pretty. There are other ways to visualize depth contour as well. Moreover, there is more information to help you find a better fishing spot. i.e. temperature, water temperature, weather, wind, water current, catch history, etc.

I want to go fishing to Lake Biwa and take a look at how this app works in practice but for now, I avoid taking a public transportation to get there due to the pandemic. I miss fishing there.

[Github](https://github.com/bathymetric-cam/bathymetric-cam-ios)