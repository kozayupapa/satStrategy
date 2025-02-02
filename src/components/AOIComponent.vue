<template>
  <!-- AOIマーカーを map 上に追加します。 -->
  <div></div>
</template>

<script lang="ts">
import { defineComponent, onMounted, onBeforeUnmount } from 'vue';
import mapboxgl from 'mapbox-gl';

export default defineComponent({
  name: 'AOIComponent',
  props: {
    map: {
      type: Object as () => mapboxgl.Map,
      required: true,
    },
    // AOI の座標 { lat, lng }
    aoiCoord: {
      type: Object as () => { lat: number; lng: number },
      required: true,
    },
  },
  setup(props) {
    let aoiMarker: mapboxgl.Marker | null = null;

    onMounted(() => {
      aoiMarker = new mapboxgl.Marker({ color: 'blue' })
        .setLngLat([props.aoiCoord.lng, props.aoiCoord.lat])
        .addTo(props.map);
    });

    onBeforeUnmount(() => {
      aoiMarker?.remove();
    });

    return {};
  },
});
</script>

<style scoped>
/* 特にスタイルは不要 */
</style>
