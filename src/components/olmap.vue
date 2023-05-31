<template>
  <div>
    <div id="map" style="width: 100%; height: 920px"></div>
    <div id="mouse-position" class="mouse-position"></div>
    <div class="button-container">
      <button @click="startDrawingPolygon">AREA</button>
    </div>

    <div
      style="
        position: fixed;
        z-index: 9999;
        color: red;
        font-size: 30px;
        top: 20vh;
        left: 20vw;
      "
    >
      {{ name }}</div>
  </div>
</template>

<script>
import 'ol/ol.css';
import { Map, View } from 'ol';
import OSM from 'ol/source/OSM';
import XYZ from "ol/source/XYZ";
import { format } from 'ol/coordinate';
import MousePosition from 'ol/control/MousePosition';
import TileLayer from 'ol/layer/Tile';
import TileWMS from 'ol/source/TileWMS';
import { Vector as VectorSource } from 'ol/source';
import { Draw } from 'ol/interaction';
import { getArea } from 'ol/sphere';
import { fromExtent } from 'ol/geom/Polygon';
import { Select } from 'ol/interaction';
import Style from 'ol/style/Style';
import Fill from 'ol/style/Fill';
import Stroke from 'ol/style/Stroke';
import OlExtLayerSwitcher from 'ol-ext/control/LayerSwitcher';
import { OverviewMap, defaults as defaultControls } from "ol/control";
import VectorLayer from "ol/layer/Vector";
import * as Format from "ol/format";
import ucas from "../../resources/ucas.json";
export default {
  name: 'OpenLayersMap',
  data() {
    return {
      map: null,
      points: [],
      mousePosition: null,
      drawnPolygon: null,
      drawInteraction: null,
      popup: null,
      tileLayer: null, // 添加tileLayer属性
    };
  },
  methods: {
    createMap() {
      let overviewMapControl = new OverviewMap({
        className: "ol-overviewmap ol-custom-overviewmap",
        layers: [
          new TileLayer({
            source: new XYZ({
            url: "http://wprd0{1-4}.is.autonavi.com/appmaptile?lang=zh_cn&size=1&style=7&x={x}&y={y}&z={z}",
            }),
          }),
        ],
        view: new View({
          projection: "EPSG:4326", //坐标系
        }),
        collapseLabel: "»", //鹰眼展开时功能按钮上的标识
        label: "«", //鹰眼折叠时功能按钮上的标识
        collapsed: false, //初始为展开显示方式
      });
      this.map = new Map({
        target: 'map',
        layers: [
          new TileLayer({
            title:'OpenStreetMap',
            source: new OSM({}),
          }),
        ],
        view: new View({
          center: [116.68, 40.41],
          projection: 'EPSG:4326',
          zoom: 16,
          minZoom: 6,
          maxZoom: 20,
        }),
      });
      this.map.addControl(overviewMapControl);
      const ctrl = new OlExtLayerSwitcher({
                      collapsed: true,
                      trash: true,
                  });
      this.map.addControl(ctrl);
      // this.popup = new Overlay({
      //   element: document.getElementById('popup'),
      //   autoPan: true,
      //   autoPanAnimation: {
      //     duration: 250,
      //   },
      // });
      // this.map.addOverlay(this.popup);
    },
    addMousePositionControl() {
      const mousePositionControl = new MousePosition({
        coordinateFormat: function (coordinate) {
          return format(coordinate, '经度:{x} 纬度:{y}', 4);
        },
        projection: 'EPSG:4326',
        className: 'custom-mouse-position',
        target: document.getElementById('mouse-position'),
        undefinedHTML: '&nbsp;',
      });
      this.map.addControl(mousePositionControl);
    },
    addLayers() {
      const layer1 = new TileLayer({
        title:'天地图矢量图',
        source: new XYZ({
          url:'http://t4.tianditu.com/DataServer?T=vec_w&x={x}&y={y}&l={z}&tk=61aa1b056de7db1906d3b43d30a71f15'
        })
      })
      const layer2 = new TileLayer({
        title:'天地图影像图',
        source: new XYZ({
          url:'http://t0.tianditu.com/DataServer?T=img_w&x={x}&y={y}&l={z}&tk=61aa1b056de7db1906d3b43d30a71f15'
        })
      })
      this.map.addLayer(layer1)
      this.map.addLayer(layer2)
    },
    loadGeoserverMap() {
      this.tileLayer = new TileLayer({ // 使用this.tileLayer进行赋值
        title:'OSM',
        source: new TileWMS({
          url: 'http://localhost:8080/geoserver/UCAS/wms',
          params: {
            LAYERS: 'UCAS:OSMUCAS',
            TILED: true,
            // STYLES: 'my-style'
          },
          serverType: 'geoserver',
        }),
      });
      this.wmsLayer=this.tileLayer;
      this.map.addLayer(this.tileLayer);
      const selectInteraction = new Select({
        layers: [this.tileLayer], // 使用this.tileLayer
      });
      this.map.addInteraction(selectInteraction);
      selectInteraction.on('select', this.onFeatureSelect);
    },
    onFeatureSelect(event) {
      const selectedFeatures = event.target.getFeatures();
      selectedFeatures.forEach((feature) => {
        feature.setStyle(this.getHighlightedStyle());
      });
    },
    getHighlightedStyle() {
      return new Style({
        stroke: new Stroke({
          color: 'yellow',
          width: 3,
        }),
        fill: new Fill({
          color: 'rgba(255, 255, 0, 0.2)',
        }),
      });
    },
    startDrawingPolygon() {
      this.drawInteraction = new Draw({
        source: new VectorSource(),
        type: 'Polygon',
      });
      this.map.addInteraction(this.drawInteraction);
      this.drawInteraction.on('drawend', this.onDrawEnd);
    },
    onDrawEnd(event) {
      const feature = event.feature;
      const geometry = feature.getGeometry();
      const coordinates = geometry.getCoordinates();
      console.log('Polygon coordinates:', coordinates);

      const area = getArea(geometry, { projection: 'EPSG:4326', radius: 6378137 });
      const areaInSquareMeters = area / 1000000;
      let output;
      if (area > 10000) {
        output = (Math.round(areaInSquareMeters * 100) / 100) + ' km<sup>2</sup>';
      } else {
        output = (Math.round(areaInSquareMeters * 100) / 100) + ' m<sup>2</sup>';
      }
      console.log('Polygon area:', output);

      const extent = geometry.getExtent();
      const polygon = fromExtent(extent);
      const center = polygon.getInteriorPoint().getCoordinates();
      console.log('Polygon center:', center);

      alert(`多边形面积：${area.toFixed(2)} 平方米`);

      this.map.removeInteraction(this.drawInteraction);
      this.drawInteraction = null;
    },
    clickFeature() {
      this.map.on("dblclick", async (e) => {
        let viewResolution = this.map.getView().getResolution();
        let url = this.wmsLayer
          .getSource()
          .getFeatureInfoUrl(e.coordinate, viewResolution, "EPSG:4326", {
            INFO_FORMAT: "application/json", //输出为json字符串
          });
        if (url) {
          let data = await fetch(url).then(function (res) {
            return res.text(); //返回Promise
          });
          if (Array.isArray(JSON.parse(data).features) && JSON.parse(data).features.length > 0) {
          console.log("JSON.parse(data):", JSON.parse(data));
          this.name = JSON.parse(data).features[0].properties.name;
          alert(this.name);
            } 
            else{
          alert(`wrong positin`);}
        }

      });
    },
    // readjson() {
    //     const geojsonFormat = new Format.GeoJSON();

    //     // Read features from GeoJSON and transform coordinates to EPSG:4326
    //     const features = geojsonFormat.readFeatures(ucas, {
    //       featureProjection: 'EPSG:3857', // Assuming the original projection is EPSG:3857
    //       dataProjection: 'EPSG:4326' // Convert coordinates to EPSG:4326
    //     });
    //     // Create a vector source with the transformed features
    //     this.boundarySource = new VectorSource({
    //       features: features,
    //     });

    //     // Create a vector layer with the transformed source and style
    //     this.boundaryLayer = new VectorLayer({
    //       source: this.boundarySource,
    //       style: new Style({
    //         stroke: new Stroke({
    //           width: 3,
    //           color: "#0f9ce2",
    //         }),
    //       }),
    //     });
    //   // Add the layer to the map
    //   this.map.addLayer(this.boundaryLayer);
    // }
  },
  mounted() {
    this.createMap();
    this.addLayers();
    this.addMousePositionControl();
    this.loadGeoserverMap();
    this.clickFeature();
    // this.readjson();
  },
};
</script>

<style scoped>
.mouse-position {
  position: absolute;
  bottom: 10px;
  left: 200px;
  background-color: rgba(255, 255, 255, 0.8);
  padding: 5px;
  border-radius: 5px;
  font-size: 14px;
}
.button-container {
  position: absolute;
  top: 60px;
  left: 10px;
}
</style>