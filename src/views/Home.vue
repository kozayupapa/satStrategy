<template>
  <div class="home">
    <div class="flex items-center justify-center space-x-8"></div>
    <SimulationComponent />
  </div>
</template>

<script lang="ts">
import { defineComponent } from "vue";
import SimulationComponent from "@/components/SimulationComponent.vue";

export default defineComponent({
  name: "HomePage",
  components: {
    SimulationComponent,
  },
});
</script>
