---
layout: page
title: Installing an OpenStreetMap Tile Server on Ubuntu
comments: true
permalink: /tile-server-ubuntu/
rendering-note: this page is best viewed with Jekyll rendering
---

## Introduction

The following step-by-step procedure can be used to install and configure all the necessary software to operate your own OpenStreetMap tile server on Ubuntu 16.4 or 14.4.[^1]

The OSM tile server stack is a collection of programs and libraries chained together to create a tile server. As so often with OpenStreetMap, there are many ways to achieve this goal and nearly all of the components have alternatives that have various specific advantages and disadvantages. This tutorial describes the most standard version that is also used on the main OpenStreetMap.org tile server.

It consists of the following main components:

* Mapnik
* Apache
* Mod_tile
* renderd
* osm2pgsql
* PostgreSQL/PostGIS database, to be installed locally (suggested) or remotely (might be slow, depending on the network).
* carto
* openstreetmap-carto

Mod_tile is an apache module that serves cached tiles and decides which tiles need re-rendering either because they are not yet cached or because they are outdated. Renderd provides a priority queueing system for rendering requests to manage and smooth out the load from rendering requests. Mapnik is the software library that does the actual rendering and is used by renderd.

A description of the rendering process of OpenStreetMap can be found at [OSM architecture](../rendering-process), including general components and tools, populating the PostGIS instance, converting the CartoCSS style sources to Mapnik XML and the Mapnik rendering process.

The process to generate tiles is the following:

|                   | |client browser  ![web][web]| | |
|                   | |↓                   | | |
|Disk Cache (tiles)![png][png]| |Apache Web Server ![prg][prg]|→|Web page ![html][html]|
|                   |↘|↓                   | | |
|renderd ![prg][prg]|←|mod_tile ![prg][prg]|←|tiles ![png][png]|
|  ↑                |↘|                      |↗|
|Mapnik XML ![xml][xml]|→|Mapnik ![prg][prg]| | |
|                      |↗|↑| | |
|PostgreSQL PostGIS ![db][db]| |shapefiles *data* directory ![shape][shape]| | |
{: .drawing}

<br />


Even if different operating system versions have been tested, Ubuntu 16.4 is strongly suggested.

{% include_relative _includes/update-ubuntu.md %}

{% include_relative _includes/unix-user.md %}

{% include_relative _includes/install-git.md program='OpenStreetMap Tile Server' %}

{% include_relative _includes/install-mapnik.md %}

{% include_relative _includes/firewall.md port='80 and local port 443' cdprogram='~/src' %}

{% include_relative _includes/install-apache.md %}

You can test Apache by accessing it through a browser at *http://your-server-ip*.

## Install Mod_tile

[Mod_tile](http://wiki.openstreetmap.org/wiki/Mod_tile) is an Apache module to efficiently render and serve map tiles for www.openstreetmap.org map using Mapnik. We can compile it from Github repository.

    test -d ~/src || mkdir  ~/src ; cd ~/src
    git clone https://github.com/openstreetmap/mod_tile.git
    cd mod_tile
    ./autogen.sh && ./configure && make && sudo make install && sudo make install-mod_tile && sudo ldconfig
    cd ~/

The rendering process implemented by mod_tile and renderd is well explained [here](https://github.com/openstreetmap/mod_tile).

{% include_relative _includes/inst-osm-carto.md cdprogram='~/src' %}

## Install carto and build the Mapnik xml stylesheet

Notice that the *carto* feature able to natively process *project.mml* in YAML format (currently adopted for openstreetmap-carto) is recent.

Install node-carto:

    sudo apt-get install -y node-carto

Ensure that the latest *carto* version is installed (at least version >= 0.16.0, using YAML):

    carto -v
    carto 0.16.3 (Carto map stylesheet compiler)

Test *carto* and produce *style.xml* from the *openstreetmap-carto* style:

    cd ~/src
    cd openstreetmap-carto
    carto -a "3.0.0" project.mml > style.xml

The option `-a "3.0.0"` is needed when using Mapnik 3 functions [^2].

{% include_relative _includes/configuration-variables.md os='Ubuntu' %}

{% include_relative _includes/postgis-inst.md port='80 and local port 443' cdprogram='~/src' %}

## Configure *renderd*

Next we need to plug renderd and mod_tile into the Apache webserver, ready to receive tile requests.

Edit *renderd* configuration file with your preferite editor:

    sudo vi /usr/local/etc/renderd.conf

In the `[default]` section, change the value of XML and HOST to the following.

    XML=/home/{{ pg_login }}/src/openstreetmap-carto/style.xml
    HOST=localhost

We suppose in the above example that your home directory is */home/{{ pg_login }}*. Change it to your actual home directory.

In `[mapnik]` section, change the value of `plugins_dir`.

    plugins_dir=/usr/lib/mapnik/3.0/input/

Pay attention to the mapnik version: if it is at version 2.2 (Ubuntu 14.4):

    plugins_dir=/usr/lib/mapnik/2.2/input/

Save the file.

Install *renderd* init script by copying the sample init script included in its package.

    sudo cp ~/src/mod_tile/debian/renderd.init /etc/init.d/renderd

Grant execute permission.

    sudo chmod a+x /etc/init.d/renderd

Edit the init script file

    sudo vi /etc/init.d/renderd

Change the following variables:

    DAEMON=/usr/local/bin/$NAME
    DAEMON_ARGS="-c /usr/local/etc/renderd.conf"
    RUNASUSER={{ pg_login }}

In `RUNASUSER={{ pg_login }}` we suppose that your user is *{{ pg_login }}*. Change it to your actual user name.

Save the file.

Create the following file and set *{{ pg_login }}* (your actual user) the owner.

    sudo mkdir -p /var/lib/mod_tile

    sudo chown {{ pg_login }}:{{ pg_login }} /var/lib/mod_tile

Again change it to your actual user name.

Then start renderd service

    sudo systemctl daemon-reload
    
    sudo systemctl start renderd

    sudo systemctl enable renderd

The following output is regular:

    renderd.service is not a native service, redirecting to systemd-sysv-install
    Executing /lib/systemd/systemd-sysv-install enable renderd

If *systemctl* is not installed (e.g., Ubuntu 14.4) use these commands respectively:

    sudo update-rc.d renderd defaults
    
    sudo service renderd start

## Configure Apache

Create a module load file.

    sudo vi /etc/apache2/mods-available/mod_tile.load

Paste the following line into the file.

    LoadModule tile_module /usr/lib/apache2/modules/mod_tile.so

Create a symlink.

    sudo ln -s /etc/apache2/mods-available/mod_tile.load /etc/apache2/mods-enabled/

Then edit the default virtual host file.

    sudo vi /etc/apache2/sites-enabled/000-default.conf

Past the following lines after the line `<VirtualHost *:80>`

```shell
LoadTileConfigFile /usr/local/etc/renderd.conf
ModTileRenderdSocketName /var/run/renderd/renderd.sock
# Timeout before giving up for a tile to be rendered
ModTileRequestTimeout 0
# Timeout before giving up for a tile to be rendered that is otherwise missing
ModTileMissingRequestTimeout 2000
```

Note: the default settings, which are the following, should already be a good tuning:

    ModTileRequestTimeout 3
    ModTileMissingRequestTimeout 30

Save and close the file. Restart Apache.

    sudo systemctl restart apache2

If *systemctl* is not installed (e.g., Ubuntu 14.4):

    sudo service apache2 restart

Test access to tiles locally:

    wget --spider http://localhost/osm_tiles/0/0/0.png

You should get `Remote file exists.` if everything is correctly configured.

Then in your web browser address bar, type

    your-server-ip/osm_tiles/0/0/0.png

where you need to change *your-server-ip* with the actual IP address of the installed map server.

You should see the tile of world map.

Congratulations! You just successfully built your own OSM tile server.

## Pre-rendering tiles

To pre-render tiles instead of rendering on the fly, use *render_list* command. Pre-rendered tiles will be cached in */var/lib/mod_tile directory*.

To show all command line option of render_list:

    render_list --help

Example usage:

    render_list -a

## Debugging Apache, mod_tile and renderd

To clear all osm tiles cache, remove /var/lib/mod_tile/default (using rm -rf if you dare) and restart renderd daemon:

    rm -rf /var/lib/mod_tile/default
    sudo systemctl restart renderd

If *systemctl* is not installed (e.g., Ubuntu 14.4):

    sudo service renderd restart

Show Apache loaded modules:

    apache2ctl -M

You should find `tile_module (shared)`

Show Apache configuration:

    apache2ctl -S

You should get the following messages within the log:

    Loading tile config default at /osm_tiles/ for zooms 0 - 20 from tile directory /var/lib/mod_tile with extension .png and mime type image/png

Tail log:

    tail -f /var/log/apache2/error.log

Most of the configuration issues can discovered by analyzing the debug log of *renderd*; we need to stop the daemon and start *renderd* in foreground:

    sudo systemctl stop renderd

If *systemctl* is not installed (e.g., Ubuntu 14.4):

    sudo service renderd stop

Then:

    /usr/local/bin/renderd -fc /usr/local/etc/renderd.conf

Ignore the five errors related to `iniparser: syntax error in /usr/local/etc/renderd.conf` when referring to commented out variables (e.g., beginning with `;`).

Press Control-C to kill the program. After fixing the error, the daemon can be restarted with:

    sudo systemctl start renderd

If *systemctl* is not installed (e.g., Ubuntu 14.4):

    sudo service renderd start

If everything in the configuration looks fine, but the map is still not rendered without any particular message produced by *renderd*, try performing a system restart:

    sudo shutdown -r now

If the problem persists, you might have a problem with your UNIX user. Try debugging again, after setting these variables:

{% include_relative _includes/configuration-variables.md os='Ubuntu' notitle='yes' %}

As exceptional case, the following commands allow to fully remove Apache, mod_tile and renderd and reinstall the service:

    sudo rm -r ~/src/mod_tile/
    sudo apt-get purge apache2 apache2-dev
    sudo rm -r /etc/apache2/mods-available
    sudo rm /usr/local/etc/renderd.conf
    sudo rm  /etc/init.d/renderd
    sudo rm -rf /var/lib/mod_tile
    sudo rm -rf /usr/lib/apache2
    sudo rm -rf /etc/apache2/
    sudo rm -rf /var/run/renderd
    sudo apt-get --reinstall install apache2-bin
    sudo apt-get install apache2 apache2-dev

## Tile names format of OpenStreetMap tile server

The file naming and image format used by mod_tile is described at [Slippy map tilenames](http://wiki.openstreetmap.org/wiki/Slippy_map_tilenames). Similar format is also used by [Google Maps](https://developers.google.com/maps/documentation/javascript/maptypes) and many other map providers.

[TMS](http://wiki.openstreetmap.org/wiki/TMS) and [WMS](http://wiki.openstreetmap.org/wiki/WMS) are other protocols for serving maps as tiles, managed by different rendering backends.

## Deploying your own Slippy Map

[Tiled web map](https://en.wikipedia.org/wiki/Tiled_web_map) is also known as [slippy map](http://wiki.openstreetmap.org/wiki/Slippy_Map) in OpenStreetMap terminology.

OpenStreetMap does not provide an “official” JavaScript library which you are required to use. Rather, you can use any library that meets your needs. The two most popular are OpenLayers and Leaflet. Both are open source.

Page [Deploying your own Slippy Map](http://wiki.openstreetmap.org/wiki/Deploying_your_own_Slippy_Map) illustrates how to embed the previously installed map server into a website. A number of possible map libraries are mentioned, including some relevant ones ([Leaflet](leafletjs.com), [OpenLayers](openlayers.org), [Google Maps API](https://developers.google.com/maps/)) as well as many alternatives.

### OpenLayers

To display your slippy map with OpenLayers, create a file named *ol.html* under */var/www/html*.

    sudo vi /var/www/html/ol.html

Paste the following HTML code into the file.

You might wish to adjust the longitude, latitude and zoom level according to your needs. Check `var zoom = 2, center = [0, 0];`.

```html
<!DOCTYPE html>
<html>
<head>
<title>OpenStreetMap with OpenLayers</title>
<link rel="stylesheet" href="https://openlayers.org/en/v3.19.0/css/ol.css" type="text/css">
<script src="https://openlayers.org/en/v3.19.0/build/ol.js"></script>
<script src="https://ajax.googleapis.com/ajax/libs/jquery/2.1.4/jquery.min.js"></script>
  <style>
  html,
  body,
  #map {
    height: 100%;
    margin: 0;
    padding: 0;
  }
  .ol-custom-overviewmap,
  .ol-custom-overviewmap.ol-uncollapsible {
    bottom: auto;
    left: auto;
    right: 0;
    top: 85px;
  }  
  .ol-zoom {
    top: 50px;
  }
  .ol-zoom-extent {
      top: 110px;
  }
  .ol-zoomslider {
      top: 140px;
  }
  .ol-custom-fullscreen {
    bottom: auto;
    left: auto;
    right: 0;
    top: 50px;
  }
  .ol-custom-mouse-positionXY {
    top: auto;
    bottom: 3em;
    font-family: "Arial";
    font-size: 12px;
    text-shadow: 0 0 0.5em #FFE, 0 0 0.5em #FFE, 0 0 0.5em #FFE;
  }
  .ol-custom-mouse-positionHDMS {
    top: auto;
    bottom: 4em;
    font-family: "Arial";
    font-size: 12px;
    text-shadow: 0 0 0.5em #FFE, 0 0 0.5em #FFE, 0 0 0.5em #FFE;
  }
  .ol-custom-mouse-position3857 {
    top: auto;
    bottom: 5em;
    font-family: "Arial";
    font-size: 12px;
    text-shadow: 0 0 0.5em #FFE, 0 0 0.5em #FFE, 0 0 0.5em #FFE;
  }
  #ZoomElement {
    position: absolute;
    top: auto;
    left: 10px;
    bottom: 2.5em;
    text-decoration: none;
    font-family: "Arial";
    font-size: 10pt;
    text-shadow: 0 0 0.5em #FFE, 0 0 0.5em #FFE, 0 0 0.5em #FFE;
    z-index: 30;
  }
  #TSLabel {
    position: absolute;
    top: 21px;
    right: 0;
    font-family: "Arial";
    font-size: 12px;
    z-index: 30;
  }
  #osmLabel {
    position: absolute;
    top: 21px;
    left: 0;
    font-family: "Arial";
    font-size: 12px;
    z-index: 30;
  }
  #swipe {
    position: absolute;
    top: 0;
    left: -4px;
    z-index: 20;
  }
</style>
</head>
<body>
  <div class="ol-viewport">
  <input class="ol-unselectable ol-control" id="swipe" type="range" style="width: 100%">
  <div class="ol-unselectable ol-control" id="TSLabel"> Tile Server &#9658;</div>
  <div class="ol-unselectable ol-control" id="osmLabel">&#9668; OpenStreetMap </div>
  <a class="ol-unselectable ol-control" id="ZoomElement"></a>
  </div>
  <div tabindex="0" id="map" class="map"></div>
  <script>
    var zoom = 2, center = [0, 0];

    // Set up the Tile Server layer
    var myTileServer = new ol.layer.Tile({
      preload: Infinity,
      source: new ol.source.OSM({
        crossOrigin: null,
        url: 'osm_tiles/{z}/{x}/{y}.png'
      })
    });
    
    // Set up the OSM layer
    var openStreetMap = new ol.layer.Tile({
      preload: Infinity,
      source: new ol.source.OSM({
        crossOrigin: null,
        url: 'http://{a-c}.tile.openstreetmap.org/{z}/{x}/{y}.png'
      })
    });

    if (window.location.hash !== '') {
      var hash = window.location.hash.replace('#', '');
      var parts = hash.split(';');
      if (parts.length === 3) {
        zoom = parseInt(parts[0], 10);
        center = [
          parseFloat(parts[2]),
          parseFloat(parts[1])
          ];
      }
    }

    // Set up the default view
    var myTileView = new ol.View({
      center: ol.proj.transform(center, 'EPSG:4326', 'EPSG:3857'),
      zoom: zoom
    });
    
    // Create the map
    var map = new ol.Map({
      layers: [myTileServer, openStreetMap],
      loadTilesWhileInteracting: true,
      target: 'map',
      controls: ol.control.defaults().extend([
        new ol.control.ScaleLine(),
        new ol.control.Zoom(),
        new ol.control.ZoomSlider(),
        new ol.control.ZoomToExtent(),
        new ol.control.FullScreen({
          className: 'ol-fullscreen ol-custom-fullscreen'
        }),
        new ol.control.OverviewMap({
          className: 'ol-overviewmap ol-custom-overviewmap'
        }),
        new ol.control.MousePosition({
          className: 'ol-mouse-position ol-custom-mouse-position3857',
          coordinateFormat: ol.coordinate.createStringXY(4),
          projection: 'EPSG:3857',
          undefinedHTML: '&nbsp;'
        }),
        new ol.control.MousePosition({
          coordinateFormat: function(coord) {
            return ol.coordinate.toStringHDMS(coord);
          },
          projection: 'EPSG:4326',
          className: 'ol-mouse-position ol-custom-mouse-positionHDMS',
          target: document.getElementById('mouse-position'),
          undefinedHTML: '&nbsp;'
        }),
        new ol.control.MousePosition({
          className: 'ol-mouse-position ol-custom-mouse-positionXY',
          coordinateFormat: ol.coordinate.createStringXY(4),
          projection: 'EPSG:4326',
          undefinedHTML: '&nbsp;'
        }),
      ]),
      view: myTileView
    });
    map.on("moveend", function() {
      var view = map.getView();
      var center = ol.proj.transform(view.getCenter(), 'EPSG:3857', 'EPSG:4326');
      var zoom = view.getZoom();
      var zoomInfo = 'Zoom level = ' + zoom;
      document.getElementById('ZoomElement').innerHTML = zoomInfo;
      window.location.hash =
        view.getZoom() + ';' +
          Math.round(center[1]*1000000)/1000000 + ';' +
          Math.round(center[0]*1000000)/1000000;
    });

    var swipe = document.getElementById('swipe');
    
    openStreetMap.on('precompose', function(event) {
        var ctx = event.context;
        var width = ctx.canvas.width * (swipe.value / 100);

        ctx.save();
        ctx.beginPath();
        ctx.rect(width, 0, ctx.canvas.width - width, ctx.canvas.height);
        ctx.clip();
      });

    openStreetMap.on('postcompose', function(event) {
        var ctx = event.context;
        ctx.restore();
      });
    
    swipe.addEventListener('input', function() {
        map.render();
    }, false);
  </script>
</body>
</html>
```

Save and close the file. Now you can view your slippy map by typing the following URL in browser.

    http://your-server-ip/ol.html

<script async src="//jsfiddle.net/ircama/r3a4t201/embed/html,js,resources,css,result/"></script>

### Leaflet

Leaflet is a JavaScript library for embedding maps. It is simpler and smaller than OpenLayers.

The easiest example to display your slippy map with Leaflet consists in creating a file named *lf.html* under */var/www/html*.

    sudo vi /var/www/html/lf.html

Paste the following HTML code in the file. Replace *your-server-ip* with your IP Address and adjust the longitude, latitude and zoom level according to your needs.

```html
<!DOCTYPE html>
<html>
<head>
<title>OpenStreetMap with Leaflet</title>
<link rel="stylesheet" href="http://cdn.leafletjs.com/leaflet-0.6.4/leaflet.css" type="text/css">
<script src="http://cdn.leafletjs.com/leaflet-0.6.4/leaflet.js"></script>
<style>
  html,
  body,
  #map {
    height: 100%;
    margin: 0;
    padding: 0;
  }
</style>
</head>
<body>
  <div id="map" class="map"></div>
  <script>
    // Create the map
    var map = L.map('map').setView([45, 10], 3);
    
    // Set up the OSM layer
    L.tileLayer(
    'http://your-server-ip/osm_tiles/{z}/{x}/{y}.png'
    ).addTo(map);
  </script>
</body>
</html>
```

Save and close the file. Now you can view your slippy map by typing the following URL in the browser.

    http://your-server-ip/lf.html

A rapid way to test the slippy map is through an online source code playground like this JSFiddle template.

{% include_relative _includes/leaflet.md os='Ubuntu' %}

{% include pages/images.md %}

[^1]: sources used for this document are the following:
    
    * [switch2osm.org - Manually building a tile server (14.04)](https://switch2osm.org/serving-tiles/manually-building-a-tile-server-14-04/)
    * [OSM Wiki - Mod tile/Setup of your own tile server](http://wiki.openstreetmap.org/wiki/Mod_tile/Setup_of_your_own_tile_server)
    * [Build Your Own OpenStreetMap Tile Server on Ubuntu 16.04](https://www.linuxbabe.com/linux-server/openstreetmap-tile-server-ubuntu-16-04)
    * [OSM tile server jessie](https://wiki.debian.org/OSM/tileserver/jessie)

[^2]: [math1985's note](https://github.com/gravitystorm/openstreetmap-carto/pull/2470#issuecomment-266234112)
