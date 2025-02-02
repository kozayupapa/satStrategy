<template>
  <div class="map-container">
    <div id="map" class="map"></div>
    <div class="controls">
      <button @click="toggle3D">{{ is3D ? 'Switch to 2D' : 'Switch to 3D' }}</button>
    </div>
  </div>
</template>

<script>
import mapboxgl from 'mapbox-gl';

export default {
  name: 'MapComponent',
  props: {
    accessToken: {
      type: String,
      required: true,
    },
    orbitData: {
      type: Array,
      required: true, 
    },
  },
  data() {
    return {
      map: null,
      satelliteMarker: null,
      is3D: false,
      animationFrame: null,
      animationIndex: 0,
    };
  },
  mounted() {
    this.initializeMap();
  },
  beforeUnmount() {
    if (this.map) this.map.remove();
    if (this.animationFrame) cancelAnimationFrame(this.animationFrame);
  },
  methods: {
    initializeMap() {
      mapboxgl.accessToken = this.accessToken;

      this.map = new mapboxgl.Map({
        container: 'map',
        style: 'mapbox://styles/mapbox/satellite-v9',
        center: [0, 0],
        zoom: 2,
        projection: this.is3D ? 'globe' : 'mercator',
      });

      this.map.on('style.load', () => {
        this.map.setFog({}); // Enable atmospheric effect for 3D mode
      });

      this.addSatelliteMarker();
      this.startAnimation();
    },

    addSatelliteMarker() {
      this.satelliteMarker = new mapboxgl.Marker({ color: 'red' })
        .setLngLat([0, 0])
        .addTo(this.map);
    },

    startAnimation() {
      this.animationIndex = 0;

      const animate = () => {
        if (this.animationIndex < this.orbitData.length) {
          const { lng, lat } = this.orbitData[this.animationIndex];
          this.satelliteMarker.setLngLat([lng, lat]);
          this.map.panTo([lng, lat]);
          this.animationIndex++;
          this.animationFrame = requestAnimationFrame(animate);
        } else {
          console.log("Animation completed");
          cancelAnimationFrame(this.animationFrame);
        }
      };

      this.animationFrame = requestAnimationFrame(animate);
    },

    toggle3D() {
      this.is3D = !this.is3D;
      this.map.setProjection(this.is3D ? 'globe' : 'mercator');
    },
  },
};
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
