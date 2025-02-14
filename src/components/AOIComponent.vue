<template>
  <!-- AOI マーカーを map 上に追加します -->
  <div></div>
</template>

<script lang="ts">
import { defineComponent, onMounted, onBeforeUnmount, watch } from "vue";
import mapboxgl from "mapbox-gl";

export default defineComponent({
  name: "AOIComponent",
  props: {
    map: {
      type: Object as () => mapboxgl.Map,
      required: true,
    },
    // AOI の座標 { lat, lon }
    aoiCoord: {
      type: Object as () => { lat: number; lon: number },
      required: true,
    },
  },
  setup(props) {
    let aoiMarker: mapboxgl.Marker | null = null;

    const addMarker = () => {
      const el = document.createElement("div");
      el.className = "aoi-marker";
      // ここでスタイルを直接設定してもOK
      el.style.width = "30px";
      el.style.height = "30px";
      el.style.backgroundColor = "blue";
      el.style.border = "3px solid white";
      el.style.borderRadius = "50%";
      el.style.boxShadow = "0 0 5px rgba(0, 0, 0, 0.5)";
      if (aoiMarker) {
        aoiMarker.remove();
      }
      aoiMarker = new mapboxgl.Marker(el).setLngLat([props.aoiCoord.lon, props.aoiCoord.lat]).addTo(props.map);
      console.log("AOI marker added at:", props.aoiCoord);
    };

    onMounted(() => {
      if (props.map.isStyleLoaded()) {
        addMarker();
      } else {
        props.map.once("load", () => {
          addMarker();
        });
      }
    });

    // props.aoiCoordが変化したらマーカーを再作成
    watch(
      () => props.aoiCoord,
      () => {
        addMarker();
      },
      { deep: true },
    );
    onBeforeUnmount(() => {
      aoiMarker?.remove();
    });

    return {};
  },
});
</script>

<style scoped>
.aoi-marker {
  z-index: 1000;
}
</style>
