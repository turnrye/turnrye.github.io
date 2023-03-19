---
title: Map
permalink: /map/
---

<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.3/dist/leaflet.css"
    integrity="sha256-kLaT2GOSpHechhsozzB+flnD+zUyjE2LlfWPgU04xyI="
    crossorigin=""/>
 <!-- Make sure you put this AFTER Leaflet's CSS -->
 <script src="https://unpkg.com/leaflet@1.9.3/dist/leaflet.js"
     integrity="sha256-WBkoXOwTeyKclOHuWtc+i2uENFpDZ9YPdf5Hf+D7ewM="
     crossorigin=""></script>
 <div id="map" style="height: 600px; width: 800px"></div>
<script type="text/javascript">
var map = L.map('map').setView([35.2091252,-89.9265784], 17);
var osm = L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
    maxZoom: 19,
    attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>'
});
var terrain = L.tileLayer('https://stamen-tiles.a.ssl.fastly.net/terrain/{z}/{x}/{y}.jpg', {
    // maxZoom: 19,
    attribution: 'Map tiles by <a href="http://stamen.com">Stamen Design</a>, under <a href="http://creativecommons.org/licenses/by/3.0">CC BY 3.0</a>. Data by <a href="http://openstreetmap.org">OpenStreetMap</a>, under <a href="http://www.openstreetmap.org/copyright">ODbL</a>.'
});
var drone = L.tileLayer('/map-tiles/{z}/{x}/{y}.jpg', {
    maxZoom: 23,
    attribution: 'Copyright 2023 Ryan Turner.'
}).addTo(map);
var overlays = {
};
var baseLayers = {
    "Terrain": terrain,
    "Streets": osm,
    "Drone": drone
};
L.control.layers(baseLayers, overlays, { collapsed: false, hideSingleBase: true }).addTo(map);
function onEachFeature(feature, layer) {
    let popupContent = `<p>${feature.properties.name}</p>`;
    if (feature.properties && feature.properties.popupContent) {
        popupContent += feature.properties.popupContent;
    }
    layer.bindPopup(popupContent);
}
</script>
