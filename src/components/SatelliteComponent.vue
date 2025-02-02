<template>
  <!-- このコンポーネントは map 上に衛星マーカーと軌跡を描画します -->
  <div></div>
</template>

<script lang="ts">
import { defineComponent, onMounted, onBeforeUnmount, watch } from 'vue';
import mapboxgl from 'mapbox-gl';

export default defineComponent({
  name: 'SatelliteComponent',
  props: {
    map: {
      type: Object as () => mapboxgl.Map,
      required: true,
    },
    // 軌道データ。各要素は { lng: number, lat: number } の形式
    orbitData: {
      type: Array as () => Array<{ lng: number; lat: number }>,
      required: true,
    },
  },
  setup(props) {
    let satelliteMarker: mapboxgl.Marker | null = null;
    let animationFrame = 0;
    let animationIndex = 0;
    const orbitSourceId = 'satOrbitSource';
    const orbitLayerId = 'satOrbitLayer';

    // 軌跡（ライン）の描画
    const createOrbitLine = () => {
      // GeoJSON Feature として properties を必ず設定
      const geojson: GeoJSON.Feature<GeoJSON.LineString, {}> = {
        type: 'Feature',
        geometry: {
          type: 'LineString',
          coordinates: props.orbitData.map((point) => [point.lng, point.lat]),
        },
        properties: {},
      };

      if (props.map.getSource(orbitSourceId)) {
        // すでに存在する場合はデータ更新
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
            'line-width': 3,
          },
        });
      }
    };

    // 衛星マーカーのアニメーション
    const startAnimation = () => {
      animationIndex = 0;
      const animate = () => {
        if (animationIndex < props.orbitData.length) {
          const { lng, lat } = props.orbitData[animationIndex];
          satelliteMarker?.setLngLat([lng, lat]);
          // 必要に応じてパンさせる（コメントアウト解除可）
          // props.map.panTo([lng, lat]);
          animationIndex++;
          animationFrame = requestAnimationFrame(animate);
        } else {
          cancelAnimationFrame(animationFrame);
        }
      };
      animationFrame = requestAnimationFrame(animate);
    };

    onMounted(() => {
      // カスタムHTML要素で大きく視認しやすい衛星マーカーを作成
      const el = document.createElement('div');
      el.className = 'satellite-marker';
      satelliteMarker = new mapboxgl.Marker(el)
        .setLngLat([props.orbitData[0].lng, props.orbitData[0].lat])
        .addTo(props.map);

      // map のスタイルが読み込まれていない場合、load イベントを待つ
      if (props.map.isStyleLoaded()) {
        createOrbitLine();
        startAnimation();
      } else {
        props.map.once('load', () => {
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

    // 軌道データが変更されたら軌跡とアニメーションを更新
    watch(
      () => props.orbitData,
      () => {
        createOrbitLine();
        animationIndex = 0;
        startAnimation();
      }
    );

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
  border-radius: 50%;
  box-shadow: 0 0 5px rgba(0, 0, 0, 0.5);
}
</style>
