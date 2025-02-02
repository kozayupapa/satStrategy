<template>
  <!-- このコンポーネントは独自のDOMを持たず、map上に衛星マーカーを追加します -->
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

    const startAnimation = () => {
      animationIndex = 0;
      const animate = () => {
        if (animationIndex < props.orbitData.length) {
          const { lng, lat } = props.orbitData[animationIndex];
          satelliteMarker?.setLngLat([lng, lat]);
          // マーカーの移動に合わせて地図をパンさせる（必要に応じて）
          props.map.panTo([lng, lat]);
          animationIndex++;
          animationFrame = requestAnimationFrame(animate);
        } else {
          // アニメーション終了時の処理（必要に応じて）
          cancelAnimationFrame(animationFrame);
        }
      };
      animationFrame = requestAnimationFrame(animate);
    };

    onMounted(() => {
      satelliteMarker = new mapboxgl.Marker({ color: 'red' })
        .setLngLat([0, 0])
        .addTo(props.map);
      startAnimation();
    });

    onBeforeUnmount(() => {
      if (satelliteMarker) {
        satelliteMarker.remove();
      }
      cancelAnimationFrame(animationFrame);
    });

    // orbitData が更新された場合、再度アニメーションを開始する例
    watch(
      () => props.orbitData,
      () => {
        animationIndex = 0;
        startAnimation();
      }
    );

    return {};
  },
});
</script>

<style scoped>
/* 特にスタイルは不要 */
</style>
