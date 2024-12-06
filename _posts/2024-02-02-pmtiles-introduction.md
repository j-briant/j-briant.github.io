---
title: PMTiles exploration
date: 2024-02-02 10:30:00
categories: [Discussion, Introduction, Opensource]
tags: [pmtiles, gis, data, opensource, tiles, protomaps, cloud, pbf, gdal, geospatial]
---

It's a technology that I came across rather recently. Tile generation when maintaining a geographic oriented website can be a hassle: it can take time, generates a lot of files, and can be a critical process. PMTiles might have the potential to facilitate some of this work and we'll try to see how.

## What are PMTiles?

In itself, PMTiles is a specification for building single-file tile pyramids. It's open source and part of a larger project called [Protomaps](https://protomaps.com/), and it aims at mapping the World, just that. The major difference with the kind of tiles that I know is that they are embedded as a single file, like the MBTile format, which make the interaction very intuitive, I mean we all get the idea of what a file is. But what does it bring compared to MBTiles? Well from the Protomaps website:

> A MBTiles file needs to be accessed as a SQLite database on disk, so requires a running server and disk storage for cloud hosting. **PMTiles is designed to be cloud-native** and readable in fine-grained parts from remote storage like Amazon S3.

It aims at simplifying the manipulation of tiles in a cloud native approach, for worldwide coverage.

## How to create PMTiles?

The [Protomaps website](https://docs.protomaps.com/pmtiles/create) gives us some tools to create these tiles. Let's try some possibilities.

### Generate PMTiles using GDAL

Our [Swiss Army Knife](https://www.geothings.ch/posts/gdal-appreciation/) is back, starting version 3.8.0 GDAL has native support for PMTiles, so let's try that.

First check the GDAL version you're using, you'll want 3.8.0 or higher:

```sh
gdalinfo --version
--> GDAL 3.8.3, released 2024/01/04 (debug build)
```

Let's start from an OSM `.pbf` file (see [Serving vector tiles with Martin](https://www.geothings.ch/posts/vector-tiles-with-martin/#downloading-some-data)). To convert this file into a PMTiles file you can enter:

```sh
ogr2ogr -f PMTiles out.pmtiles switzerland-latest.osm.pbf
```

The process might take some time if you're working on a large dataset. By default the MAXZOOM is 5, which is rather low, so the run shouldn't take to long, but if you start to increase the parameter to around 15, which allows to represent data from much closer, processing time increases.

But once the command is done, well that's it!

Alternatively we can use [tippecanoe](https://github.com/felt/tippecanoe) to achieve the same result.

### Generate PMTiles using tippecanoe

> Under construction
{: .prompt-info }

## Visualize PMTiles

The easiest way is to go to the [PMTiles Viewer Page](https://protomaps.github.io/PMTiles/) and upload your newly created PMTiles file. 

Obviously you can add it to your custom project. The easiest way of doing is by using the Leaflet plugin [protomaps-leaflet](https://github.com/protomaps/protomaps-leaflet).

Starting from complete scratch you can create an `.html` file with the following content:

```html
<!DOCTYPE html>
<html>

<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.7.1/dist/leaflet.css" />
    <script src="https://unpkg.com/leaflet@1.7.1/dist/leaflet.js"></script>
    <script src="https://unpkg.com/protomaps-leaflet@latest/dist/protomaps-leaflet.min.js"></script>
    <style>
        body,
        #map {
            height: 100vh;
            margin: 0px;
        }
    </style>
</head>

<body>

    <h2>Hey I'm a custom page</h2>
    <p>Let's add some Leaflet and PMTiles.</p>
    <div id="map"></div>

    <script>
        let PAINT_RULES = [
            {
                dataLayer: "multipolygons", //here we render only one layer, check your layers names 
                symbolizer: new protomapsL.PolygonSymbolizer({ fill: "steelblue" })
            }
        ];

        const map = L.map('map').setView([46.5, 6.6], 13); //chose a view that suits you

        var osm = L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
            maxZoom: 19,
            attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>'
        }).addTo(map);

        var layer = protomapsL.leafletLayer(
            {
                url: 'http://127.0.0.1:8080/your_pmtile_file.pmtiles',
                paint_rules: PAINT_RULES
            }).addTo(map);

    </script>

</body>
</html>
```

For testing purposes you might have to serve you PMTiles through http. To do so you can do as the Protomaps website suggests:

```sh
npm install -g http-server
# Then from the folder containing your PMTiles.
http-server . --cors
```

Open your `.html` file on the browser of your choice and you should see the map with your PMTiles displayed.

## When should PMTiles be used?

Again, the documentation on [Protomaps website](https://protomaps.com/faq) is quite complete and gives us some insights. 

### Cloud hosting

PMTiles are stored as a single file, and compression is so efficient that the size is ridiculous compared to classic png tiles. Number of requests necessary is drastically reduced and thus the cost of operation as well.

### Static data

PMTiles don't support localized updates so each time you'll need to apply an update, the whole file needs to be rewritten. If your data don't change that much PMTiles are a wise choice, but If you need to have a more dynamic rendering the documentation recommends PostGIS with the [ST_As_MVT()](https://postgis.net/docs/ST_AsMVT.html) function.

### Simplicity 

Here it's more like a feeling about the philosophy of the specification. You have a single file, accessible without introduced complexity, directly from a library like Leaflet, which itself has _simplicity_ as one of its core objectives. It looks like one of those project that allows you to make complicated-looking things so easily. I guess it's a sign of a good project.

## Final words

Well it was rather easy to discover. I mean we create a file, we basically load it within a project and we're good to go. Obviously there are some customing to do with styling and labels etc. But once you have the starting point and your tools, it takes minutes to create what you desire.

Here we're just scratching the surface, remains to explore filtering, styling etc. But the concept in itself and the unreasonnable performance of the compression make it very interesting if you're in the situation where PMTiles could give you an advantage; the file created for OSM data Switzerland, with zooms from 0 to 14, is around 1GB, it costs pennies to upload and host on cloud services.

It's a project to support and look for, I guess?