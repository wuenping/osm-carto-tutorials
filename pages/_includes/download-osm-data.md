## Get an OpenStreetMap data extract

You need to download an appropriate .osm or .pbf file to be subsequently loaded into the previously created PostGIS instance via `osm2pgsql`.

There are many ways to download the OSM data.

The reference is [Planet OSM](http://planet.openstreetmap.org).

It's probably easiest to grab an PBF of OSM data from [Mapzen](https://mapzen.com/metro-extracts/) or [geofabrik](http://download.geofabrik.de/).

Also, [BBBike.org](http://download.bbbike.org/osm/) provides extracts of more than 200 cities and regions world-wide in different formats.

Examples:

* Map data of the whole planet (32G): <http://planet.openstreetmap.org/pbf/planet-latest.osm.pbf>
* Map data of Great Britain (847M): <wget -c http://download.geofabrik.de/europe/great-britain-latest.osm.pbf>

One method is directly with your browser. Check [this page](http://wiki.openstreetmap.org/wiki/Downloading_data).

Alternatively, [JOSM](https://josm.openstreetmap.de/) can be used (Select the area to download the OSM data: JOSM menu, File, Download From OSM; tab Slippy map; drag the map with the right mouse button, zoom with the mouse wheel or Ctrl + arrow keys; drag a box with the left mouse button to select an area to download. The [Continuous Download](http://wiki.openstreetmap.org/wiki/JOSM/Plugins/continuosDownload) plugin is also suggested. When the desired region is locally available, select File, Save As, `<filename>.osm`. Give it a valid file name and check also the appropriate directory where this file is saved.