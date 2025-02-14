<template>
  <div class="map-container">
    <div id="map" class="map"></div>
    <!-- マップインスタンスが生成されてから子コンポーネントをレンダリング -->
    <div v-if="map">
      <SatelliteComponent v-for="(sat, index) in satellites" :key="index" :map="map as any" :orbit-data="sat.orbitData" />
      <AOIComponent v-for="(aoi, index) in aois" :key="index" :map="map as any" :aoi-coord="aoi" />
    </div>
    <div class="controls">
      <button @click="toggle3D">
        {{ is3D ? "Switch to 2D" : "Switch to 3D" }}
      </button>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, onMounted, onBeforeUnmount } from "vue";
import mapboxgl from "mapbox-gl";
import SatelliteComponent from "./SatelliteComponent.vue";
import AOIComponent from "./AOIComponent.vue";

interface SatelliteOrbit {
  orbitData: Array<{ lat: number; lng: number }>;
}

interface AOI {
  lat: number;
  lon: number;
}

export default defineComponent({
  name: "MapComponent",
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

    onMounted(() => {
      mapboxgl.accessToken = props.accessToken;
      const m = new mapboxgl.Map({
        container: "map",
        style: "mapbox://styles/mapbox/satellite-v9",
        center: [0, 0],
        zoom: 1.5,
        projection: "globe",
        pitch: 60,
        bearing: 0,
      });
      if (m) {
        m.on("style.load", () => {
          map.value?.setFog({
            color: "rgba(0,0,0,0)",
            "high-color": "#f8f0e3",
            "space-color": "#d8e2dc",
            range: [1, 12],
            "horizon-blend": 0.5,
          });
        });
      }
      map.value = m;
    });

    onBeforeUnmount(() => {
      if (map.value) map.value.remove();
    });

    const toggle3D = () => {
      is3D.value = !is3D.value;
      map.value?.setProjection(is3D.value ? "globe" : "mercator");
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
