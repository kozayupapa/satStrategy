<template>
  <div class="map-container">
    <div id="map" class="map"></div>
    <!-- マップインスタンスが生成されてから子コンポーネントをレンダリング -->
     <div  v-if="map">
    <SatelliteComponent
     
      v-for="(sat, index) in satellites"
      :key="index"
      :map="map as any"
      :orbit-data="sat.orbitData"
    />
    <AOIComponent
      v-for="(aoi, index) in aois"
      :key="index"
      :map="map as any"
      :aoi-coord="aoi"
    />
    </div>
    <div class="controls">
      <button @click="toggle3D">
        {{ is3D ? 'Switch to 2D' : 'Switch to 3D' }}
      </button>
    </div>
  </div>
</template>


<script lang="ts">
import { defineComponent, ref, onMounted, onBeforeUnmount } from 'vue';
import mapboxgl from 'mapbox-gl';
import SatelliteComponent from './SatelliteComponent.vue';
import AOIComponent from './AOIComponent.vue';
import * as SunCalc from 'suncalc'

interface SatelliteOrbit {
  orbitData: Array<{ lat: number; lng: number }>;
}

interface AOI {
  lat: number;
  lon: number;
}

export default defineComponent({
  name: 'MapComponent',
  components: { SatelliteComponent, AOIComponent },
  props: {
    accessToken: {
      type: String,
      required: true,
    },
    satellites: {
      type: Array as () => SatelliteOrbit[],
      required: true,
    },
    aois: {
      type: Array as () => AOI[],
      required: true,
    },
  },
  setup(props) {
    const map = ref<mapboxgl.Map | null>(null);
    const is3D = ref(false);

    // 現在の太陽の位置を取得して、マップのライト設定に反映する関数
    const updateSunPosition = () => {
      const now = new Date();
      // マップ中心の緯度経度（例として 0,0 を使用。実際は任意の地点に合わせる）
      const centerLat = 0;
      const centerLon = 0;
      // SunCalc.getPosition(時刻, 緯度, 経度)
      const sunPos = SunCalc.getPosition(now, centerLat, centerLon);
      console.log(sunPos);
      // sunPos.azimuth は南から時計回り（ラジアン）、sunPos.altitude は太陽高度（ラジアン）
      // Mapbox の light.position は [azimuth (deg), altitude (deg), 0] として設定（その他のパラメータは intensity など）
      map.value?.setLight({
        anchor: 'viewport',
        position: [
          (sunPos.azimuth * 180) / Math.PI, // azimuth [deg]
          (sunPos.altitude * 180) / Math.PI,  // altitude [deg]
          0,
        ],
        intensity: 0.8, // 必要に応じて調整
      });

      // 定期的に更新（アニメーションとしても使えます）
      requestAnimationFrame(updateSunPosition);
    };

    onMounted(() => {
      mapboxgl.accessToken = props.accessToken;

      // 地球（globe）表示用のマップを作成
      map.value = new mapboxgl.Map({
        container: 'map',
        style: 'mapbox://styles/mapbox/satellite-v9',
        center: [0, 0],
        zoom: 1.5,
        projection: 'globe',  // globe 表示にする
      });
      map.value?.on('style.load', () => {
        map.value?.setFog({
          color: 'rgba(0,0,0,0)',           // fog の基本色（ここでは透明に）
          'high-color': '#f8f0e3',           // 高高度の光の色
          'space-color': '#d8e2dc',          // 宇宙側の色
          range: [1, 12],
          'horizon-blend': 0.5,
        });
        // 太陽の位置に基づいてマップのライトを更新するループ開始
        updateSunPosition();
      });
    });

    onBeforeUnmount(() => {
      if (map.value) map.value.remove();
    });


    const toggle3D = () => {
      is3D.value = !is3D.value;
      map.value?.setProjection(is3D.value ? 'globe' : 'mercator');
    };

    return { map, is3D, toggle3D };
  },
});
</script>

<style scoped>
.map-container {
  position: relative;
  width: 100%;
  height: 100%;
}
.map {
  width: 100%;
  height: 100vh;
}
.controls {
  position: absolute;
  top: 10px;
  left: 10px;
  z-index: 10;
}
button {
  background: rgba(255, 255, 255, 0.8);
  border: none;
  padding: 10px;
  border-radius: 5px;
  cursor: pointer;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
}
</style>
