<template>
  <div class="map-container">
    <div id="map" class="map"></div>
    <!-- map が初期化されてから子コンポーネントを描画 -->
    <SatelliteComponent
      v-if="map"
      :map="map"
      :orbitData="orbitData"
    />
    <AOIComponent
      v-if="map"
      :map="map"
      :aoiCoord="aoiCoord"
    />
    <div class="controls">
      <button @click="toggle3D">
        {{ is3D ? 'Switch to 2D' : 'Switch to 3D' }}
      </button>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, onMounted, onBeforeUnmount, ref } from 'vue';
import mapboxgl from 'mapbox-gl';
import SatelliteComponent from './SatelliteComponent.vue';
import AOIComponent from './AOIComponent.vue';

export default defineComponent({
  name: 'MapComponent',
  components: { SatelliteComponent, AOIComponent },
  props: {
    accessToken: {
      type: String,
      required: true,
    },
    // 軌道データ。各要素は { lng: number, lat: number } の形式
    orbitData: {
      type: Array as () => Array<{ lng: number; lat: number }>,
      required: true,
    },
    // AOI の座標。 { lat: number, lng: number }
    aoiCoord: {
      type: Object as () => { lat: number; lng: number },
      required: true,
    },
  },
  setup(props) {
    const map = ref<mapboxgl.Map | null>(null);
    const is3D = ref(false);

    onMounted(() => {
      mapboxgl.accessToken = props.accessToken;

      map.value = new mapboxgl.Map({
        container: 'map',
        style: 'mapbox://styles/mapbox/satellite-v9',
        center: [0, 0],
        zoom: 2,
        projection: is3D.value ? 'globe' : 'mercator',
      });

      map.value.on('style.load', () => {
        map.value?.setFog({}); // 3Dモード時の大気表現など
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
