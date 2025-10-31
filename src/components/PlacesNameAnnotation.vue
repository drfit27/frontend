<template>
  <div class="place-name-annotation">
    <!-- 控制面板 -->
    <div class="control-panel" v-if="showControls">
      <h3>地名标注工具</h3>
      <button @click="startAdding">添加标注</button>
      <button @click="clearAll">清除所有</button>
      <button @click="toggleVisibility">
        {{ showAnnotations ? '隐藏标注' : '显示标注' }}
      </button>
      <div v-if="isPicking" class="hint">请在地图上单击选择位置…</div>
      <div class="search-by-address">
        <input
          v-model="addressQuery"
          placeholder="输入地址，例如：北京天安门"
          @keyup.enter="searchBaiduAndAdd"
        />
        <button @click="searchBaiduAndAdd">地址搜索并添加</button>
      </div>
      <div class="annotation-list" v-if="annotations.length > 0">
        <h4>已添加的标注 ({{ annotations.length }})</h4>
        <div 
          v-for="(annotation, index) in annotations" 
          :key="index"
          class="annotation-item"
        >
          <span>{{ annotation.name }}</span>
          <button @click="removeAnnotation(index)">删除</button>
        </div>
      </div>
    </div>

    <!-- 添加标注的模态框 -->
    <div v-if="isAdding" class="add-modal">
      <div class="modal-content">
        <h3>添加地名标注</h3>
        <input 
          v-model="newAnnotationName" 
          placeholder="输入地名"
          @keyup.enter="confirmAdd"
        />
        <div class="modal-actions">
          <button @click="confirmAdd">确认</button>
          <button @click="cancelAdd">取消</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import * as Cesium from 'cesium'

export default {
  name: 'PlaceNameAnnotation',
  props: {
    // 接收 Cesium viewer 实例
    viewer: {
      type: Object,
      required: true
    },
    // 是否显示控制面板
    showControls: {
      type: Boolean,
      default: true
    }
  },
  data() {
    return {
      isAdding: false,
      isPicking: false,
      newAnnotationName: '',
      annotations: [],
      showAnnotations: true,
      clickHandler: null,
      temporaryPoint: null,
      addressQuery: ''
    }
  },
  emits: ['annotation-added'],
  mounted() {
    this.initializeAnnotations();
    // 监听来自 Viewer 的地理编码结果事件
    window.addEventListener('geocode-result', this.onGeocodeResult)
  },
  beforeUnmount() {
    window.removeEventListener('geocode-result', this.onGeocodeResult)
    this.cleanup();
  },
  methods: {
    initializeAnnotations() {
      // 从本地存储加载已保存的标注
      const saved = localStorage.getItem('cesium_annotations');
      if (saved) {
        this.annotations = JSON.parse(saved);
        this.renderAnnotations();
      }
    },

    onGeocodeResult() {},

    // BD09 -> WGS84 转换（先 BD09 -> GCJ02，再 GCJ02 -> WGS84）
    bd09ToGcj02(bdLon, bdLat) {
      const x = bdLon - 0.0065;
      const y = bdLat - 0.006;
      const z = Math.sqrt(x * x + y * y) - 0.00002 * Math.sin(y * Math.PI);
      const theta = Math.atan2(y, x) - 0.000003 * Math.cos(x * Math.PI);
      const gcjLon = z * Math.cos(theta);
      const gcjLat = z * Math.sin(theta);
      return { gcjLon, gcjLat };
    },
    gcj02ToWgs84(gcjLon, gcjLat) {
      // 简化版近似反算（适合标注场景），需要更高精度可替换为迭代反解
      const d = this.delta(gcjLat, gcjLon);
      return {
        wgsLon: gcjLon * 2 - d.lon,
        wgsLat: gcjLat * 2 - d.lat
      };
    },
    transformLat(x, y) {
      let ret = -100.0 + 2.0 * x + 3.0 * y + 0.2 * y * y + 0.1 * x * y + 0.2 * Math.sqrt(Math.abs(x));
      ret += (20.0 * Math.sin(6.0 * x * Math.PI) + 20.0 * Math.sin(2.0 * x * Math.PI)) * 2.0 / 3.0;
      ret += (20.0 * Math.sin(y * Math.PI) + 40.0 * Math.sin(y / 3.0 * Math.PI)) * 2.0 / 3.0;
      ret += (160.0 * Math.sin(y / 12.0 * Math.PI) + 320 * Math.sin(y * Math.PI / 30.0)) * 2.0 / 3.0;
      return ret;
    },
    transformLon(x, y) {
      let ret = 300.0 + x + 2.0 * y + 0.1 * x * x + 0.1 * x * y + 0.1 * Math.sqrt(Math.abs(x));
      ret += (20.0 * Math.sin(6.0 * x * Math.PI) + 20.0 * Math.sin(2.0 * x * Math.PI)) * 2.0 / 3.0;
      ret += (20.0 * Math.sin(x * Math.PI) + 40.0 * Math.sin(x / 3.0 * Math.PI)) * 2.0 / 3.0;
      ret += (150.0 * Math.sin(x / 12.0 * Math.PI) + 300.0 * Math.sin(x / 30.0 * Math.PI)) * 2.0 / 3.0;
      return ret;
    },
    outOfChina(lat, lon) {
      return lon < 72.004 || lon > 137.8347 || lat < 0.8293 || lat > 55.8271;
    },
    delta(lat, lon) {
      const a = 6378245.0;
      const ee = 0.00669342162296594323;
      if (this.outOfChina(lat, lon)) return { lat, lon };
      let dLat = this.transformLat(lon - 105.0, lat - 35.0);
      let dLon = this.transformLon(lon - 105.0, lat - 35.0);
      const radLat = lat / 180.0 * Math.PI;
      let magic = Math.sin(radLat);
      magic = 1 - ee * magic * magic;
      const sqrtMagic = Math.sqrt(magic);
      dLat = (dLat * 180.0) / ((a * (1 - ee)) / (magic * sqrtMagic) * Math.PI);
      dLon = (dLon * 180.0) / (a / sqrtMagic * Math.cos(radLat) * Math.PI);
      const mgLat = lat + dLat;
      const mgLon = lon + dLon;
      return { lat: mgLat, lon: mgLon };
    },

    async searchBaiduAndAdd() {
      const query = this.addressQuery.trim()
      console.log('[PlaceNameAnnotation] searchBaiduAndAdd clicked, query=', query)
      if (!query) {
        console.warn('[PlaceNameAnnotation] empty query')
        return
      }
      try {
        const ak = 'hxQigLG0cMPhOwMKoQZDUt8vGHutAehO'
        const url = `https://api.map.baidu.com/geocoding/v3/?address=${encodeURIComponent(query)}&output=json&ak=${ak}`
        console.log('[PlaceNameAnnotation] fetching:', url)
        const resp = await fetch(url, { headers: { 'Accept': 'application/json' } })
        console.log('[PlaceNameAnnotation] response status:', resp.status)
        const data = await resp.json().catch(() => null)
        console.log('[PlaceNameAnnotation] response json:', data)
        if (!data || data.status !== 0 || !data.result || !data.result.location) {
          console.warn('[PlaceNameAnnotation] baidu no result or error status', data && data.status)
          return
        }
        const bdLon = Number(data.result.location.lng)
        const bdLat = Number(data.result.location.lat)
        console.log('[PlaceNameAnnotation] baidu BD09 coords:', { bdLon, bdLat })
        const { gcjLon, gcjLat } = this.bd09ToGcj02(bdLon, bdLat)
        const { wgsLon, wgsLat } = this.gcj02ToWgs84(gcjLon, gcjLat)
        console.log('[PlaceNameAnnotation] converted coords:', { wgsLon, wgsLat })
        const name = data.result.level || query
        this.addAnnotationFromSearch(name, wgsLon, wgsLat)
      } catch (e) {
        console.error('[PlaceNameAnnotation] baidu search error', e)
      }
    },

    startAdding() {
      console.log('[PlaceNameAnnotation] startAdding clicked')
      this.isPicking = true;
      this.newAnnotationName = '';
      
      // 监听地图点击事件
      this.clickHandler = new Cesium.ScreenSpaceEventHandler(this.viewer.scene.canvas);
      this.clickHandler.setInputAction((event) => {
        const scene = this.viewer.scene;
        const windowPos = event.position;
        let position = null;
        if (scene.pickPositionSupported) {
          position = scene.pickPosition(windowPos);
        }
        if (!position) {
          position = this.viewer.camera.pickEllipsoid(windowPos, scene.globe.ellipsoid);
        }
        if (position) {
          this.addTemporaryPoint(position);
          this.isAdding = true;
          this.isPicking = false;
          console.log('[PlaceNameAnnotation] map clicked, temporary point set')
        }
      }, Cesium.ScreenSpaceEventType.LEFT_CLICK);
    },

    addTemporaryPoint(position) {
      // 清除之前的临时点
      if (this.temporaryPoint) {
        this.viewer.entities.remove(this.temporaryPoint);
      }

      // 添加临时点标记
      this.temporaryPoint = this.viewer.entities.add({
        position: position,
        point: {
          pixelSize: 10,
          color: Cesium.Color.YELLOW,
          outlineColor: Cesium.Color.BLACK,
          outlineWidth: 2,
          heightReference: Cesium.HeightReference.CLAMP_TO_GROUND
        }
      });
    },

    async confirmAdd() {
      console.log('[PlaceNameAnnotation] confirmAdd clicked, name=', this.newAnnotationName)
      if (!this.newAnnotationName.trim() || !this.temporaryPoint) {
        console.warn('[PlaceNameAnnotation] confirmAdd aborted: missing name or temporaryPoint')
        return;
      }
      const now = Cesium.JulianDate.now();
      const cartesian = this.temporaryPoint.position.getValue(now);
      if (!cartesian) return;
      const cartographic = Cesium.Cartographic.fromCartesian(cartesian);
      const longitude = Cesium.Math.toDegrees(cartographic.longitude);
      const latitude = Cesium.Math.toDegrees(cartographic.latitude);

      const annotation = {
        id: Date.now(),
        name: this.newAnnotationName.trim(),
        longitude: longitude,
        latitude: latitude
      };

      this.annotations.push(annotation);
      this.saveAnnotations();
      this.renderAnnotations();

      // 通知父组件
      this.$emit('annotation-added', annotation);

      // 清理临时状态
      this.cancelAdd();
      console.log('[PlaceNameAnnotation] confirmAdd success, annotation=', annotation)
    },
   
    addAnnotationFromSearch(name, longitude, latitude) {
      console.log('[PlaceNameAnnotation] addAnnotationFromSearch', { name, longitude, latitude })
      const annotation = {
        id: Date.now(),
        name,
        longitude,
        latitude
      };

      this.annotations.push(annotation);
      this.saveAnnotations();
      this.renderAnnotations();

      this.viewer.camera.flyTo({
        destination: Cesium.Cartesian3.fromDegrees(longitude, latitude, 2000)
      });

      this.$emit('annotation-added', annotation);
      this.addressQuery = '';
      return annotation;
    },

    cancelAdd() {
      this.isAdding = false;
      this.isPicking = false;
      this.newAnnotationName = '';
      
      if (this.clickHandler) {
        this.clickHandler.destroy();
        this.clickHandler = null;
      }
      
      if (this.temporaryPoint) {
        this.viewer.entities.remove(this.temporaryPoint);
        this.temporaryPoint = null;
      }
    },

    renderAnnotations() {
      // 清除所有标注实体
      this.viewer.entities.removeAll();

      if (!this.showAnnotations) return;

      this.annotations.forEach(annotation => {
        const position = Cesium.Cartesian3.fromDegrees(annotation.longitude, annotation.latitude);
        this.viewer.entities.add({
          id: `annotation_${annotation.id}`,
          position: position,
          label: {
            text: annotation.name,
            font: '16px sans-serif',
            fillColor: Cesium.Color.WHITE,
            backgroundColor: Cesium.Color.BLACK.withAlpha(0.7),
            backgroundPadding: new Cesium.Cartesian2(8, 8),
            pixelOffset: new Cesium.Cartesian2(0, -30),
            scale: 1.0,
            showBackground: true,
            heightReference: Cesium.HeightReference.CLAMP_TO_GROUND,
            horizontalOrigin: Cesium.HorizontalOrigin.CENTER,
            verticalOrigin: Cesium.VerticalOrigin.BOTTOM
          }
        });
      });
    },

    removeAnnotation(index) {
      this.annotations.splice(index, 1);
      this.saveAnnotations();
      this.renderAnnotations();
    },

    clearAll() {
      this.annotations = [];
      this.saveAnnotations();
      this.renderAnnotations();
    },

    toggleVisibility() {
      this.showAnnotations = !this.showAnnotations;
      this.renderAnnotations();
    },

    saveAnnotations() {
      localStorage.setItem('cesium_annotations', JSON.stringify(this.annotations));
    },

    cleanup() {
      if (this.clickHandler) {
        this.clickHandler.destroy();
      }
      if (this.temporaryPoint) {
        this.viewer.entities.remove(this.temporaryPoint);
      }
    }
  }
}
</script>

<style scoped>
.place-name-annotation {
  position: relative;
}

.control-panel {
  position: fixed;
  top: 10px;
  left: 10px;
  background: rgba(0, 0, 0, 0.8);
  color: white;
  padding: 15px;
  border-radius: 8px;
  z-index: 1000;
  max-width: 300px;
}

.control-panel button {
  display: block;
  width: 100%;
  margin: 5px 0;
  padding: 8px;
  background: #0078ff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.control-panel button:hover {
  background: #0056cc;
}

.annotation-list {
  margin-top: 15px;
  border-top: 1px solid #555;
  padding-top: 10px;
}

.annotation-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 5px 0;
  border-bottom: 1px solid #333;
}

.annotation-item button {
  width: auto;
  padding: 3px 8px;
  background: #ff4444;
  font-size: 12px;
}

.annotation-item button:hover {
  background: #cc0000;
}

.add-modal {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 2000;
}

.modal-content {
  background: white;
  padding: 20px;
  border-radius: 8px;
  min-width: 300px;
}

.modal-content input {
  width: 100%;
  padding: 8px;
  margin: 10px 0;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.modal-actions {
  display: flex;
  gap: 10px;
}

.modal-actions button {
  flex: 1;
  padding: 8px;
}
</style>