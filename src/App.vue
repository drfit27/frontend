<template>
  <div id="app">
    <!-- Cesium 地图容器 -->
    <CesiumViewer 
      ref="cesiumViewer"
      @viewer-ready="onViewerReady"
    />
    
    <!-- 地名标注组件 -->
    <PlaceNameAnnotation 
      v-if="viewer"
      :viewer="viewer"
      :show-controls="true"
      :tianditu-token="tiandituToken"
      @annotation-added="onAnnotationAdded"
    />
  </div>
</template>

<script>
import CesiumViewer from './components/CesiumViewer.vue'
import PlaceNameAnnotation from './components/PlacesNameAnnotation.vue'

export default {
  name: 'App',
  components: {
    CesiumViewer,
    PlaceNameAnnotation
  },
  data() {
    return {
      viewer: null,
      
    }
  },
  methods: {
    onViewerReady(viewer) {
      this.viewer = viewer
      console.log('Cesium Viewer 已初始化')
    },
    
    onAnnotationAdded(annotation) {
      console.log('标注已添加:', annotation)
      // 可以在这里处理标注添加后的业务逻辑
    }
  }
}
</script>

<style>
#app {
  width: 100%;
  height: 100vh;
  margin: 0;
  padding: 0;
  position: relative;
}
</style>
