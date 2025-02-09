<template>
  <!-- このコンポーネントは map 上に衛星マーカーと軌跡を描画します -->
  <div></div>
</template>

<script lang="ts">
import { defineComponent, onMounted, onBeforeUnmount, watch } from 'vue';
import mapboxgl from 'mapbox-gl';
import type { Feature, LineString } from 'geojson';

export default defineComponent({
  name: 'SatelliteComponent',
  props: {
    map: {
      type: Object as () => mapboxgl.Map,
      required: true,
    },
    // 軌道データ。各要素は { lat: number, lng: number } の形式
    orbitData: {
      type: Array as () => Array<{ lat: number; lng: number }>,
      required: true,
    },
  },
  setup(props) {
    let satelliteMarker: mapboxgl.Marker | null = null;
    let animationFrame = 0;
    let animationIndex = 0;
    // 複数インスタンス同士が衝突しないようにランダムなソース／レイヤー ID を生成
    const orbitSourceId = `satOrbitSource-${Math.random().toString(36).substring(2, 7)}`;
    const orbitLayerId = `satOrbitLayer-${Math.random().toString(36).substring(2, 7)}`;

    const createOrbitLine = () => {
      // GeoJSON Feature として properties を必ず設定
      const geojson: Feature<LineString, Record<string, unknown>> = {
        type: "Feature", // リテラル "Feature" を指定
        geometry: {
          type: "LineString", // リテラル "LineString" を指定
          coordinates: props.orbitData.map(point => [point.lng, point.lat]),
        },
        properties: {},
      };
      if (props.map.getSource(orbitSourceId)) {
        (props.map.getSource(orbitSourceId) as mapboxgl.GeoJSONSource).setData(geojson);
      } else {
        props.map.addSource(orbitSourceId, {
          type: 'geojson',
          data: geojson,
        });
        props.map.addLayer({
          id: orbitLayerId,
          type: 'line',
          source: orbitSourceId,
          layout: {
            'line-join': 'round',
            'line-cap': 'round',
          },
          paint: {
            'line-color': '#FF0000',
            'line-width': 1,
          },
        });
      }
    };

    const startAnimation = () => {
      animationIndex = 0;
      const animate = () => {
        if (animationIndex < props.orbitData.length) {
          const { lng, lat } = props.orbitData[animationIndex];
          satelliteMarker?.setLngLat([lng, lat]);
          animationIndex++;
          animationFrame = requestAnimationFrame(animate);
        } else {
          animationIndex=0;
          animationFrame = requestAnimationFrame(animate);
        }
      };
      animationFrame = requestAnimationFrame(animate);
    };

    const addMarker = () => {
      const el = document.createElement('div');
      el.className = 'satellite-marker';
      el.style.width = '30px';
      el.style.height = '30px';
      el.style.backgroundColor = 'red';
      el.style.border = '1px solid white';
      el.style.borderRadius = '50%';
      el.style.boxShadow = '0 0 5px rgba(0, 0, 0, 0.5)';

      satelliteMarker = new mapboxgl.Marker(el)
        .setLngLat([props.orbitData[0].lng, props.orbitData[0].lat])
        .addTo(props.map);
    };

    onMounted(() => {

      if (props.map.isStyleLoaded()) {
        addMarker();
        createOrbitLine();
        startAnimation();
      } else {
        props.map.once('load', () => {
          addMarker();
          createOrbitLine();
          startAnimation();
        });
      }
    });

    onBeforeUnmount(() => {
      satelliteMarker?.remove();
      cancelAnimationFrame(animationFrame);
      if (props.map.getLayer(orbitLayerId)) {
        props.map.removeLayer(orbitLayerId);
      }
      if (props.map.getSource(orbitSourceId)) {
        props.map.removeSource(orbitSourceId);
      }
    });

    watch(() => props.orbitData, () => {
      createOrbitLine();
      animationIndex = 0;
      startAnimation();
    });

    return {};
  },
});
</script>

<style scoped>
.satellite-marker {
  width: 30px;
  height: 30px;
  background-color: red;
  border: 3px solid white;
  z-index: 1000;
  border-radius: 50%;
  box-shadow: 0 0 5px rgba(0, 0, 0, 0.5);
}
</style>
