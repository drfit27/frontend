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
        <h4>地址搜索</h4>
        <div class="search-input-wrapper">
          <input
            v-model="addressQuery"
            placeholder="输入地址搜索，例如：北京天安门"
            @keyup.enter="searchBaiduAndAdd"
            :disabled="isSearching"
          />
          <button 
            @click="searchBaiduAndAdd" 
            :disabled="isSearching || !addressQuery.trim()"
            class="search-button"
          >
            {{ isSearching ? '搜索中...' : '搜索并定位' }}
          </button>
        </div>
        <div v-if="searchError" class="error-message">{{ searchError }}</div>
        <div v-if="searchSuccess" class="success-message">{{ searchSuccess }}</div>
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
      addressQuery: '',
      isSearching: false,
      searchError: '',
      searchSuccess: ''
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

    // 带超时的fetch请求
    async fetchWithTimeout(url, options = {}, timeout = 10000) {
      const controller = new AbortController()
      const id = setTimeout(() => controller.abort(), timeout)
      
      try {
        const response = await fetch(url, {
          ...options,
          signal: controller.signal
        })
        clearTimeout(id)
        return response
      } catch (error) {
        clearTimeout(id)
        if (error.name === 'AbortError') {
          throw new Error('请求超时，请检查网络连接')
        }
        throw error
      }
    },

    // 使用Nominatim（OpenStreetMap）作为备用服务
    async searchNominatim(query) {
      const url = `https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(query)}&limit=1&accept-language=zh-CN,zh,en`
      console.log('[PlaceNameAnnotation] Trying Nominatim:', url)
      
      try {
        const resp = await this.fetchWithTimeout(url, {
          headers: {
            'Accept': 'application/json',
            'User-Agent': 'CesiumMapApp/1.0'
          }
        }, 8000)
        
        if (!resp.ok) {
          throw new Error(`Nominatim请求失败: ${resp.status}`)
        }
        
        const data = await resp.json()
        console.log('[PlaceNameAnnotation] Nominatim response:', data)
        
        if (!data || data.length === 0) {
          throw new Error('未找到该地址')
        }
        
        const result = data[0]
        const longitude = parseFloat(result.lon)
        const latitude = parseFloat(result.lat)
        const name = result.display_name || query
        
        return { name, longitude, latitude, source: 'nominatim' }
      } catch (e) {
        console.error('[PlaceNameAnnotation] Nominatim error:', e)
        throw e
      }
    },

    // 百度地图搜索
    async searchBaidu(query) {
      const ak = 'hxQigLG0cMPhOwMKoQZDUt8vGHutAehO'
      const url = `https://api.map.baidu.com/geocoding/v3/?address=${encodeURIComponent(query)}&output=json&ak=${ak}`
      console.log('[PlaceNameAnnotation] Trying Baidu:', url)
      
      try {
        const resp = await this.fetchWithTimeout(url, {
          headers: { 'Accept': 'application/json' }
        }, 8000)
        
        if (!resp.ok) {
          throw new Error(`百度API请求失败: ${resp.status}`)
        }
        
        const data = await resp.json()
        console.log('[PlaceNameAnnotation] Baidu response:', data)
        
        if (!data || data.status !== 0 || !data.result || !data.result.location) {
          const errorMsg = data?.message || '未找到该地址'
          throw new Error(errorMsg)
        }
        
        const bdLon = Number(data.result.location.lng)
        const bdLat = Number(data.result.location.lat)
        
        // BD09 -> WGS84 坐标转换
        const { gcjLon, gcjLat } = this.bd09ToGcj02(bdLon, bdLat)
        const { wgsLon, wgsLat } = this.gcj02ToWgs84(gcjLon, gcjLat)
        
        const name = data.result.formatted_address || data.result.level || query
        
        return { name, longitude: wgsLon, latitude: wgsLat, source: 'baidu' }
      } catch (e) {
        console.error('[PlaceNameAnnotation] Baidu error:', e)
        throw e
      }
    },

    async searchBaiduAndAdd() {
      const query = this.addressQuery.trim()
      console.log('[PlaceNameAnnotation] searchBaiduAndAdd clicked, query=', query)
      if (!query) {
        this.searchError = '请输入要搜索的地址'
        setTimeout(() => { this.searchError = '' }, 3000)
        return
      }

      // 重置状态
      this.isSearching = true
      this.searchError = ''
      this.searchSuccess = ''

      try {
        let result = null
        let lastError = null
        
        // 先尝试百度地图
        try {
          result = await this.searchBaidu(query)
          console.log('[PlaceNameAnnotation] Baidu search success')
        } catch (baiduError) {
          console.warn('[PlaceNameAnnotation] Baidu failed, trying fallback:', baiduError)
          lastError = baiduError
          
          // 如果百度失败，尝试使用Nominatim作为备用
          try {
            result = await this.searchNominatim(query)
            console.log('[PlaceNameAnnotation] Nominatim search success')
          } catch (nominatimError) {
            console.error('[PlaceNameAnnotation] All services failed')
            lastError = nominatimError
            throw lastError
          }
        }
        
        if (!result) {
          throw new Error('搜索失败，请稍后重试')
        }
        
        // 添加标注并飞到位置
        const annotation = this.addAnnotationFromSearch(result.name, result.longitude, result.latitude)
        
        const sourceText = result.source === 'baidu' ? '百度地图' : 'OpenStreetMap'
        this.searchSuccess = `成功定位到: ${result.name} (${sourceText})`
        setTimeout(() => { this.searchSuccess = '' }, 5000)
        
        return annotation
        
      } catch (e) {
        console.error('[PlaceNameAnnotation] search error:', e)
        
        // 提供更友好的错误提示
        let errorMessage = '搜索失败'
        if (e.message.includes('超时')) {
          errorMessage = '请求超时，请检查网络连接'
        } else if (e.message.includes('网络') || e.message.includes('Failed to fetch') || e.message.includes('Network')) {
          errorMessage = '网络连接失败，请检查您的网络设置'
        } else if (e.message.includes('未找到')) {
          errorMessage = '未找到该地址，请尝试其他关键词或更详细的地址'
        } else {
          errorMessage = `搜索失败: ${e.message}`
        }
        
        this.searchError = errorMessage
        setTimeout(() => { this.searchError = '' }, 7000)
      } finally {
        this.isSearching = false
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

      // 飞到该位置，使用合适的视角高度
      this.viewer.camera.flyTo({
        destination: Cesium.Cartesian3.fromDegrees(longitude, latitude, 2000),
        orientation: {
          heading: Cesium.Math.toRadians(0),
          pitch: Cesium.Math.toRadians(-45),
          roll: 0.0
        },
        duration: 2.0 // 飞行动画时长2秒
      });

      this.$emit('annotation-added', annotation);
      // 不清空搜索框，让用户可以继续搜索其他地址
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

.search-by-address {
  margin-top: 15px;
  padding-top: 15px;
  border-top: 1px solid #555;
}

.search-by-address h4 {
  margin: 0 0 10px 0;
  font-size: 14px;
  color: #fff;
}

.search-input-wrapper {
  display: flex;
  gap: 8px;
  margin-bottom: 8px;
}

.search-input-wrapper input {
  flex: 1;
  padding: 8px;
  border: 1px solid #555;
  border-radius: 4px;
  background: rgba(255, 255, 255, 0.1);
  color: white;
  font-size: 14px;
}

.search-input-wrapper input::placeholder {
  color: rgba(255, 255, 255, 0.5);
}

.search-input-wrapper input:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.search-button {
  padding: 8px 16px;
  white-space: nowrap;
}

.search-button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
  background: #555 !important;
}

.error-message {
  color: #ff6b6b;
  font-size: 12px;
  margin-top: 5px;
  padding: 5px;
  background: rgba(255, 107, 107, 0.1);
  border-radius: 4px;
}

.success-message {
  color: #51cf66;
  font-size: 12px;
  margin-top: 5px;
  padding: 5px;
  background: rgba(81, 207, 102, 0.1);
  border-radius: 4px;
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