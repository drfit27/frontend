<!-- CesiumViewer.vue -->
<template>
  <div id="cesiumContainer" ref="viewerDiv" />
</template>

<script setup>
import { onMounted, ref, defineEmits } from 'vue'
import * as Cesium from 'cesium'
import 'cesium/Build/Cesium/Widgets/widgets.css'

const viewerDiv = ref(null)
const token = '0937241431a95cefd9d638c24285d0ec'    
const emit = defineEmits(['viewer-ready'])

onMounted(() => {
  const viewer = new Cesium.Viewer(viewerDiv.value, {
    baseLayerPicker: false,
    geocoder: false,
    homeButton: true,
    timeline: false,
    navigationHelpButton: false,
    animation: false,
    fullscreenButton: false,
    vrButton: false,
  })
  
  // 天地图影像
  viewer.imageryLayers.addImageryProvider(
    new Cesium.UrlTemplateImageryProvider({
      url: `https://t{s}.tianditu.gov.cn/DataServer?T=img_w&x={x}&y={y}&l={z}&tk=${token}`,
      subdomains: ['0', '1', '2', '3', '4', '5', '6', '7'],
      tilingScheme: new Cesium.WebMercatorTilingScheme(),
      maximumLevel: 18,
    })
  )

  // 天地图国界
  viewer.imageryLayers.addImageryProvider(
    new Cesium.UrlTemplateImageryProvider({
      url: `https://t{s}.tianditu.gov.cn/DataServer?T=ibo_w&x={x}&y={y}&l={z}&tk=${token}`,
      subdomains: ['0', '1', '2', '3', '4', '5', '6', '7'],
      tilingScheme: new Cesium.WebMercatorTilingScheme(),
      maximumLevel: 10,
    })
  )
  //地形  
function addTerrain(viewer) {
  Cesium.CesiumTerrainProvider.fromUrl(
    'https://data.mars3d.cn/terrain',
    {
      requestWaterMask: true,
      requestVertexNormals: true
    }
  ).then(provider => {
    viewer.terrainProvider = provider
  }).catch(e => console.log('加载地形失败', e))
}
addTerrain(viewer)

  // 飞到中国
  viewer.camera.flyTo({
    destination: Cesium.Cartesian3.fromDegrees(103.84, 31.15, 17850000),
    orientation: {
      heading: Cesium.Math.toRadians(348.4),
      pitch: Cesium.Math.toRadians(-89.7),
      roll: 0,
    },
  })

  // 提升贴地点击的准确性
  viewer.scene.globe.depthTestAgainstTerrain = true

  // 通知父组件 Viewer 已就绪
  emit('viewer-ready', viewer)
})
</script>

<style scoped>
#cesiumContainer {
  width: 100%;
  height: 100%;
}
</style>