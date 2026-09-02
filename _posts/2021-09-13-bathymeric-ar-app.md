---
layout: post
title: Bathymetric AR App
date: 2021-09-13T17:21:03+0300
description: bathymetric-cam is an iOS AR app that visualizes water-depth contours.
tags: iOS app fishing
categories: MobileApp
giscus_comments: true
related_posts: false
toc:
  sidebar: left
---

This post introduces my side project, bathymetric-cam. It is an iOS AR app that visualizes water-depth contours.

## Motivation

Have you ever wanted to know the water depth on site?

That may sound a bit odd, but I have. For anglers especially, depth contours can be a hint for finding the target species. A depth contour is a line connecting points of equal depth on an ocean or lake floor. When the lines are spaced tightly, the bottom drops off steeply. Anglers call those spots drop-offs, and they matter when you are targeting species such as largemouth bass. Largemouth bass are also known for holding in deeper water when it is cold in winter. bathymetric-cam shows those contours in an AR view to help you find a better fishing spot.

## Prototyping

### Data Creation

To visualize water depth, I first needed data for a proof of concept. I drew the depth-contour polygons by hand in [QGIS](https://www2.qgis.org), then exported a GeoJSON file per map tile using the [slippy map tile system](https://wiki.openstreetmap.org/wiki/Slippy_map_tilenames).

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

Then I wrote a [script](https://github.com/bathymetric-cam/geojson-to-map-tile) that converts those GeoJSON files into PNG map tiles. For example, the GeoJSON below becomes the tile image next to it.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid path="/assets/img/2021-09-13-bathymeric-ar-app-geojson.jpg" title="GeoJSON" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid path="/assets/img/2021-09-13-bathymeric-ar-app-maptile.png" title="Map tile" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

I ended up drawing part of the south lake of [Lake Biwa](https://en.wikipedia.org/wiki/Lake_Biwa). Manual labor is definitely not for a lazy programmer…

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid path="/assets/img/2021-09-13-bathymeric-ar-app-qgis.webp" title="Map tiles" class="img-fluid" %}
    </div>
</div>

The PNG files are uploaded to a CDN, which works as a simplified map-tile server.

### iOS app

The prototype is a simple AR app with a rounded map at the bottom. When you turn the iPhone camera, the map turns with it, so the AR view and the map face the same direction. Both render the same tiles from the tile server. A slider sets the altitude of the water surface. The camera is the anchor; the water surface sits X meters below it.

{% include video.liquid path="https://www.youtube.com/embed/HrZpjp9iqkA" class="img-fluid rounded z-depth-1" %}

## What's next

Honestly, I am still deciding what to improve next. The jaggy tiles do not look great. There are other ways to visualize depth contours, and more information that would help you find a better spot: air temperature, water temperature, weather, wind, current, catch history, and so on.

I want to go fishing at Lake Biwa and see how the app works in practice, but for now I am avoiding public transit because of the pandemic. I miss fishing there.

[Github](https://github.com/bathymetric-cam/bathymetric-cam-ios)
