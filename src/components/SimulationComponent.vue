<template>
  <div>
    <h1>SAR Satellite Simulation</h1>
    <form @submit.prevent="startSimulation">
      <section>
        <h2>Satellites</h2>
        <div
          v-for="(sat, index) in satellites"
          :key="index"
          style="margin-bottom: 1rem; padding: 0.5rem; border: 1px solid #ccc;"
        >
          <label>
            Orbit Type:
            <select v-model="sat.orbitType">
              <option value="sun-synchronous">Sun-Synchronous</option>
              <option value="inclined">Inclined</option>
            </select>
          </label>
          <div v-if="sat.orbitType === 'inclined'">
            <label>
              Launch Inclination Angle (deg):
              <input type="number" v-model.number="sat.launchAngle" />
            </label>
          </div>
          <div>
            <label>
              Launch Location Latitude:
              <input type="number" v-model.number="sat.launchLat" />
            </label>
            <label>
              Launch Location Longitude:
              <input type="number" v-model.number="sat.launchLon" />
            </label>
          </div>
          <div>
            <label>
              Altitude (km):
              <input type="number" v-model.number="sat.altitude" />
            </label>
          </div>
        </div>
        <button type="button" @click="addSatellite">Add Satellite</button>
      </section>

      <section style="margin-top: 1rem;">
        <h2>AOI</h2>
        <div
          v-for="(aoi, index) in aois"
          :key="index"
          style="margin-bottom: 0.5rem;"
        >
          <label>
            AOI Latitude:
            <input type="number" v-model.number="aoi.lat" />
          </label>
          <label>
            AOI Longitude:
            <input type="number" v-model.number="aoi.lon" />
          </label>
        </div>
        <button type="button" @click="addAOI">Add AOI</button>
      </section>

      <div style="margin-top: 1rem;">
        <button type="submit">Start Simulation</button>
      </div>
    </form>

    <div v-if="simulationStarted" style="margin-top: 2rem;">
      <!-- マップコンポーネントへ、計算済みの各衛星軌道データと AOI 情報を渡す -->
      <MapComponent
        :access-token="accessToken"
        :satellites="computedSatellites"
        :aois="aois"
      />
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, computed } from 'vue';
import MapComponent from './MapComponent.vue';

const R_EARTH = 6371; // 地球半径 [km]
const MU = 398600; // 地球の重力定数 [km^3/s^2]
const TIME_SCALE = 1440; // 1 simulation second corresponds to 1440 real seconds (86400 sec/day ÷ 60 sec)
const SIM_DURATION = 120; // シミュレーションの総秒数（＝1分間）

// 各衛星の入力パラメータ
interface SatelliteInput {
  orbitType: 'sun-synchronous' | 'inclined';
  launchAngle: number; // 傾斜軌道の場合のみ有効（deg）
  launchLat: number;
  launchLon: number;
  altitude: number; // [km]
}

// AOI の座標
interface AOI {
  lat: number;
  lon: number;
}

// 各衛星の軌道データ（計算結果）
interface SatelliteOrbit {
  orbitData: Array<{ lat: number; lng: number }>;
}

export default defineComponent({
  name: 'SimulationComponent',
  components: { MapComponent },
  setup() {
    const accessToken = ref('MAPBOX_TOKEN_REMOVED');

    // 初期状態として 1 つの衛星、1 つの AOI を用意
    const satellites = ref<SatelliteInput[]>([
      {
        orbitType: 'sun-synchronous',
        launchAngle: 97.8, // sun-synchronous では固定
        launchLat: 0,
        launchLon: 0,
        altitude: 500, // km
      },
    ]);

    const aois = ref<AOI[]>([
      { lat: 15, lon: 15 },
    ]);

    const simulationStarted = ref(false);

    const addSatellite = () => {
      satellites.value.push({
        orbitType: 'sun-synchronous',
        launchAngle: 97.8,
        launchLat: 0,
        launchLon: 0,
        altitude: 500,
      });
    };

    const addAOI = () => {
      aois.value.push({ lat: 0, lon: 0 });
    };

   // シミュレーションの計算部分（例）
  const computedSatellites = computed<SatelliteOrbit[]>(() => {
    return satellites.value.map((sat) => {
      // 円軌道と仮定して、軌道周期を計算（実時間）
      const orbitalPeriod = 2 * Math.PI * Math.sqrt(Math.pow(R_EARTH + sat.altitude, 3) / MU);
      const orbitData: Array<{ lat: number; lng: number }> = [];

      // シミュレーションは SIM_DURATION 秒間（＝1分間）のアニメーション
      for (let t_sim = 0; t_sim <= SIM_DURATION; t_sim++) {
        const t_real = t_sim * TIME_SCALE; // 実時間 [s]
        const theta = ((t_real % orbitalPeriod) / orbitalPeriod) * 2 * Math.PI; // 角度 [rad]
        const latVariation =
          sat.orbitType === 'inclined'
            ? sat.launchAngle * Math.sin(theta)
            : 10 * Math.sin(theta);
        const lat = sat.launchLat + latVariation;
        const lonRaw = sat.launchLon + (360 * (t_real / orbitalPeriod));
        const lon = ((lonRaw + 180) % 360) - 180;
        orbitData.push({ lat, lng: lon });
      }
      return { orbitData };
    });
  });


    const startSimulation = () => {
      simulationStarted.value = true;
    };

    return {
      accessToken,
      satellites,
      aois,
      addSatellite,
      addAOI,
      startSimulation,
      simulationStarted,
      computedSatellites,
    };
  },
});
</script>

<style scoped>
form {
  max-width: 600px;
  margin: 0 auto;
}
label {
  display: block;
  margin: 0.5rem 0;
}
input,
select {
  margin-left: 0.5rem;
}
</style>
