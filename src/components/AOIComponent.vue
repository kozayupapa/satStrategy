<template>
  <!-- AOI マーカーを map 上に追加します -->
  <div></div>
</template>

<script lang="ts">
import { defineComponent, onMounted, onBeforeUnmount, ref } from 'vue';
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
    const aoiMarker = ref<mapboxgl.Marker | null>(null);
    
    const addMarker = () => {
    // マーカー用の要素を作成
    const el = document.createElement('div');
    el.className = 'aoi-marker';
    // 必要に応じてスタイルを直接指定することも可能です
    el.style.width = '30px';
    el.style.height = '30px';
    el.style.backgroundColor = 'blue';
    el.style.border = '3px solid white';
    el.style.borderRadius = '50%';
    el.style.boxShadow = '0 0 5px rgba(0, 0, 0, 0.5)';

    // すでにマーカーが存在している場合は、一度削除
    if (aoiMarker.value) {
      aoiMarker.value.remove();
    }

    // props.aoiCoord の値は { lat: number, lng: number } として渡される前提
    aoiMarker.value = new mapboxgl.Marker(el)
      .setLngLat([props.aoiCoord.lng, props.aoiCoord.lat])
      .addTo(props.map);

    console.log('AOI marker added at: ', props.aoiCoord);
  };
    onMounted(() => {
      console.log('AOIComponent mounted, aoiCoord:', props.aoiCoord);
      if (props.map.isStyleLoaded()) {
        console.log('Map style is loaded, adding AOI marker.');
        addMarker();
      } else {
        console.log('Map style not loaded, waiting for load event.');
        props.map.once('load', () => {
          console.log('Map load event fired, adding AOI marker.');
          addMarker();
        });
      }      
    });

    onBeforeUnmount(() => {
      aoiMarker.value?.remove();
    });

    return {};
  },
});
</script>

<style scoped>
.aoi-marker {
  width: 30px;
  height: 30px;
  background-color: blue;
  border: 3px solid white;
  border-radius: 50%;
  box-shadow: 0 0 5px rgba(0, 0, 0, 0.5);
  z-index: 1000;
}
</style>
