<template>
  <div>
    <h1>SAR Satellite Simulation</h1>
    <form @submit.prevent="startSimulation">
      <section>
        <h2>Satellites</h2>
        <div v-for="(sat, index) in satellites" :key="index" style="margin-bottom: 1rem; padding: 0.5rem; border: 1px solid #ccc">
          <label>
            Orbit Type:
            <select v-model="sat.orbitType">
              <option value="sun-synchronous">Sun-Synchronous</option>
              <option value="inclined">Inclined</option>
            </select>
          </label>
          <div v-if="sat.orbitType === 'sun-synchronous'">
            <label>
              Launch Inclination Angle (deg):
              <input type="number" step="any" :value="97.8" disabled />
            </label>
          </div>
          <div v-else>
            <label>
              Launch Inclination Angle (deg):
              <input type="number" step="any" v-model.number="sat.launchAngle" />
            </label>
          </div>
          <div>
            <label>
              Launch Location Latitude:
              <input type="number" step="any" v-model.number="sat.launchLat" />
            </label>
            <label>
              Launch Location Longitude:
              <input type="number" step="any" v-model.number="sat.launchLon" />
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

      <section style="margin-top: 1rem">
        <h2>AOI</h2>
        <div v-for="(aoi, index) in aois" :key="index" style="margin-bottom: 0.5rem">
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

      <div style="margin-top: 1rem">
        <button type="submit">Start Simulation</button>
      </div>
    </form>
    <!-- 撮像待ち時間の結果表示 -->
    <div v-if="imagingWaitResults.length > 0" class="results">
      <h2>Imaging Wait Times</h2>
      <table>
        <thead>
          <tr>
            <th>Satellite Index</th>
            <th>AOI Index</th>
            <th>Average Wait Time (sec)</th>
            <th>Maximum Wait Time (sec)</th>
            <th>Imaging Times (sec)</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="result in imagingWaitResults" :key="result.satelliteIndex + '-' + result.aoiIndex">
            <td>{{ result.satelliteIndex }}</td>
            <td>{{ result.aoiIndex }}</td>
            <td>{{ result.avgWait !== null ? result.avgWait.toFixed(2) : "N/A" }}</td>
            <td>{{ result.maxWait !== null ? result.maxWait.toFixed(2) : "N/A" }}</td>
            <td>{{ result.imagingTimes.join(", ") }}</td>
          </tr>
        </tbody>
      </table>
    </div>
    <div v-if="simulationStarted" style="margin-top: 2rem">
      <!-- マップコンポーネントへ、計算済みの各衛星軌道データと AOI 情報を渡す -->
      <MapComponent :access-token="accessToken" :satellites="computedSatellites" :aois="aois" />
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, computed } from "vue";
import MapComponent from "./MapComponent.vue";
import * as satellite from "satellite.js";

const R_EARTH = 6371; // 地球半径 [km]
const MU = 398600; // 地球の重力定数 [km^3/s^2]
const SIM_DURATION = 720; // シミュレーションの秒数（例：1分間で1日分をシミュレーション）
const TIME_SCALE = (60 * 60 * 24) / SIM_DURATION; // シミュレーション時刻と実時間の比（86400秒÷60秒）
export const ORBIT_TYPES = {
  SUN_SYNCHRONOUS: "sun-synchronous",
  INCLINED: "inclined",
} as const;

export type OrbitType = (typeof ORBIT_TYPES)[keyof typeof ORBIT_TYPES];
// 各衛星の入力パラメータ
interface SatelliteInput {
  orbitType: OrbitType;
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
  name: "SimulationComponent",
  components: { MapComponent },
  setup() {
    const accessToken = ref("MAPBOX_TOKEN_REMOVED");

    // 初期状態として 1 つの衛星、1 つの AOI を用意
    const satellitesRef = ref<SatelliteInput[]>([
      {
        orbitType: ORBIT_TYPES.SUN_SYNCHRONOUS,
        launchAngle: 97.8, // sun-synchronous では固定
        launchLat: 0,
        launchLon: 0,
        altitude: 500, // km
      },
    ]);

    // ここで、satellitesRef をグローバルな reactive 変数として定義\
    /*
    const satellitesRef = ref<SatelliteInput[]>([
      { orbitType: 'inclined', launchLat: 30, launchLon: 140, altitude: 500, launchAngle: 97.8 }
    ]);*/

    const aois = ref<AOI[]>([{ lat: 15, lon: 15 }]);

    const simulationStarted = ref(false);

    const addSatellite = () => {
      satellitesRef.value.push({
        orbitType: ORBIT_TYPES.SUN_SYNCHRONOUS,
        launchAngle: 97.8,
        launchLat: 0,
        launchLon: 0,
        altitude: 500,
      });
    };

    const addAOI = () => {
      aois.value.push({ lat: 0, lon: 0 });
    };

    const toRadians = (deg: number): number => {
      return (deg * Math.PI) / 180;
    };

    const toDegrees = (rad: number): number => {
      return (rad * 180) / Math.PI;
    };
    const haversineDistance = (lat1: number, lon1: number, lat2: number, lon2: number): number => {
      const R = 6371;
      const dLat = toRadians(lat2 - lat1);
      const dLon = toRadians(lon2 - lon1);
      const a = Math.sin(dLat / 2) ** 2 + Math.cos(toRadians(lat1)) * Math.cos(toRadians(lat2)) * Math.sin(dLon / 2) ** 2;
      const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
      return R * c;
    };
    /**
     * 2点 (lat1, lon1) から (lat2, lon2) までの方位角（真北から時計回り、度）を計算
     */
    const computeBearing = (lat1: number, lon1: number, lat2: number, lon2: number): number => {
      const φ1 = toRadians(lat1);
      const φ2 = toRadians(lat2);
      const Δλ = toRadians(lon2 - lon1);
      const y = Math.sin(Δλ) * Math.cos(φ2);
      const x = Math.cos(φ1) * Math.sin(φ2) - Math.sin(φ1) * Math.cos(φ2) * Math.cos(Δλ);
      let θ = Math.atan2(y, x);
      // 角度に変換し、0～360度の範囲に正規化
      θ = (toDegrees(θ) + 360) % 360;
      return θ;
    };

    /**
     * 2つの角度の差を [0, 180] の範囲で返す
     */
    const angleDifference = (a: number, b: number): number => {
      let diff = Math.abs(a - b) % 360;
      if (diff > 180) diff = 360 - diff;
      return diff;
    };

    /**
     * ユーザー入力からダミーTLEを生成する関数
     * @param sat ユーザーが入力した衛星パラメータ
     * @returns { line1: string, line2: string } ダミーTLEの2行
     */
    const generateDummyTLE = (sat: SatelliteInput): { line1: string; line2: string } => {
      // 'sun-synchronous' の場合は打ち上げ傾斜角を 97.8 固定
      const inclination = sat.orbitType === ORBIT_TYPES.SUN_SYNCHRONOUS ? 97.8 : sat.launchAngle;
      // RAAN を打ち上げ経度として簡易設定
      const raan = sat.launchLon;
      //const eccentricity = 0;       // 円軌道と仮定
      const argPerigee = 0; // 近地点引数：0
      const meanAnomaly = 0; // 平均近点角：0

      // 平均運動 (revs per day) の計算
      const semiMajorAxis = R_EARTH + sat.altitude; // km
      const n = Math.sqrt(MU / Math.pow(semiMajorAxis, 3)); // rad/s
      const meanMotion = (n * 86400) / (2 * Math.PI); // revs per day

      // Epoch の生成（TLE epoch は YYDDD.DDDDDDDD 形式）
      const now = new Date();
      const year = now.getUTCFullYear() % 100; // 下2桁
      const startOfYear = new Date(Date.UTC(now.getUTCFullYear(), 0, 1));
      const dayOfYear = Math.floor((now.getTime() - startOfYear.getTime()) / (1000 * 60 * 60 * 24)) + 1;
      const secondsOfDay = now.getUTCHours() * 3600 + now.getUTCMinutes() * 60 + now.getUTCSeconds() + now.getUTCMilliseconds() / 1000;
      const fraction = secondsOfDay / 86400;
      // Epoch を "YYDDD.DDDDDDDD" 形式に整形（ここでは小数部は 8 桁固定）
      const epoch = `${year.toString().padStart(2, "0")}${dayOfYear.toString().padStart(3, "0")}.${fraction.toFixed(8).slice(2)}`;

      // ダミーの衛星番号（"00001" で固定）
      const satNum = "00001";

      // TLE Line 1（フォーマットの厳密なチェックディジット計算などは省略）
      const line1 = `1 ${satNum}U 00000A   ${epoch}  .00000000  00000-0  00000-0 0  9991`;

      // TLE Line 2 の各項目は所定の桁数で埋める必要があります。
      // 以下は簡易フォーマット例です。
      // - Inclination: 8桁（例：" 97.8000"）
      // - RAAN: 8桁
      // - Eccentricity: 7桁（小数点なし、0の場合は"0000000"）
      // - Argument of Perigee: 8桁
      // - Mean Anomaly: 8桁
      // - Mean Motion: 11桁（小数点含む）
      const inclStr = inclination.toFixed(4).padStart(8, " ");
      const raanStr = raan.toFixed(4).padStart(8, " ");
      const eccStr = "0000000"; // ecc = 0
      const argPerigeeStr = argPerigee.toFixed(4).padStart(8, " ");
      const meanAnomalyStr = meanAnomaly.toFixed(4).padStart(8, " ");
      const meanMotionStr = meanMotion.toFixed(8).padStart(11, " ");

      const line2 = `2 ${satNum} ${inclStr} ${raanStr} ${eccStr} ${argPerigeeStr} ${meanAnomalyStr} ${meanMotionStr}`;

      return { line1, line2 };
    };

    /**
     * computedSatellites: ユーザー入力（SatelliteInput）からダミーTLEを生成し、satellite.js を利用して
     * シミュレーション時刻における衛星の地上トラック（緯度・経度の配列）を算出する computed プロパティの例
     */
    const computedSatellites = computed<SatelliteOrbit[]>(() => {
      // ここでは、satellites.value は SatelliteInput[] 型の配列とする（ユーザー入力済み）
      // ※以下は例として、入力データの配列を想定
      // 例:
      // const satellites = ref<SatelliteInput[]>([
      //   { orbitType: 'inclined', launchLat: 30, launchLon: 140, altitude: 500, launchAngle: 0 },
      //   { orbitType: 'sun-synchronous', launchLat: 0, launchLon: 0, altitude: 600, launchAngle: 5 },
      // ]);

      return satellitesRef.value.map((satInput) => {
        // ダミーTLEを生成
        const { line1, line2 } = generateDummyTLE(satInput);
        // TLE から satrec を作成（SGP4 propagator を内部的に使用）
        const satrec = satellite.twoline2satrec(line1, line2);
        const orbitData: Array<{ lat: number; lng: number }> = [];

        // シミュレーション開始時刻（現在時刻）
        const startTime = new Date();

        // SIM_DURATION ステップ分ループ
        for (let t_sim = 0; t_sim <= SIM_DURATION; t_sim++) {
          // 各ステップの実時間（秒）に換算
          const t_offset_sec = t_sim * TIME_SCALE;
          const currentTime = new Date(startTime.getTime() + t_offset_sec * 1000);

          // propagate() を用いて ECI 座標（位置、速度）を計算
          const posVel = satellite.propagate(satrec, currentTime);
          if (posVel.position && typeof posVel.position !== "boolean") {
            // ECI 座標から GMST (Greenwich Mean Sidereal Time) を計算
            const gmst = satellite.gstime(currentTime);
            // ECI 座標を地球固定座標（ジオデティック：緯度、経度、高度）に変換
            const geodetic = satellite.eciToGeodetic(posVel.position, gmst);
            // 緯度・経度を度単位に変換
            const lat = satellite.degreesLat(geodetic.latitude);
            const lon = satellite.degreesLong(geodetic.longitude);
            orbitData.push({ lat, lng: lon });
          }
        }
        return { orbitData };
      });
    });

    // imagingWaitResults: 各衛星軌道と各 AOI の組み合わせで、撮像条件を満たすタイミングを抽出し、
    // 撮像間隔から平均待ち時間と最大待ち時間を計算する
    const imagingWaitResults = computed(() => {
      const results: Array<{
        satelliteIndex: number;
        aoiIndex: number;
        avgWait: number | null;
        maxWait: number | null;
        imagingTimes: number[];
      }> = [];
      const lateralTolerance = 5; // 真横条件の許容誤差（度）
      const offNadirMin = 20; // Off-Nadir 角の下限 (度)
      const offNadirMax = 45; // Off-Nadir 角の上限 (度)
      const TIME_SCALE_LOCAL = TIME_SCALE;

      computedSatellites.value.forEach((satOrbit, satIndex) => {
        const { orbitData } = satOrbit;
        aois.value.forEach((aoi, aoiIndex) => {
          const imagingTimes: number[] = [];
          for (let i = 0; i < orbitData.length - 1; i++) {
            const currentPos = orbitData[i];
            const nextPos = orbitData[i + 1];
            const heading = computeBearing(currentPos.lat, currentPos.lng, nextPos.lat, nextPos.lng);
            const lateral1 = (heading + 90) % 360;
            const lateral2 = (heading + 270) % 360;
            const bearingToAOI = computeBearing(currentPos.lat, currentPos.lng, aoi.lat, aoi.lon);
            const diff1 = angleDifference(bearingToAOI, lateral1);
            const diff2 = angleDifference(bearingToAOI, lateral2);
            if (diff1 <= lateralTolerance || diff2 <= lateralTolerance) {
              // Off-Nadir 角の計算
              const distance = haversineDistance(currentPos.lat, currentPos.lng, aoi.lat, aoi.lon);
              // offNadir = arctan(distance / altitude) (sat.altitude in km)
              const offNadirDeg = toDegrees(Math.atan(distance / satellitesRef.value?.[satIndex].altitude));
              if (offNadirDeg >= offNadirMin && offNadirDeg <= offNadirMax) {
                imagingTimes.push(i * TIME_SCALE_LOCAL);
              }
            }
          }
          if (imagingTimes.length >= 2) {
            const intervals: number[] = [];
            for (let j = 0; j < imagingTimes.length - 1; j++) {
              intervals.push(imagingTimes[j + 1] - imagingTimes[j]);
            }
            const avgInterval = intervals.reduce((sum, dt) => sum + dt, 0) / intervals.length;
            const avgWait = avgInterval / 2;
            const maxWait = Math.max(...intervals);
            results.push({
              satelliteIndex: satIndex,
              aoiIndex: aoiIndex,
              avgWait,
              maxWait,
              imagingTimes,
            });
          } else {
            results.push({
              satelliteIndex: satIndex,
              aoiIndex: aoiIndex,
              avgWait: null,
              maxWait: null,
              imagingTimes,
            });
          }
        });
      });
      return results;
    });
    const startSimulation = () => {
      simulationStarted.value = true;
    };

    return {
      accessToken,
      addSatellite,
      addAOI,
      satellites: satellitesRef,
      aois,
      simulationStarted,
      startSimulation,
      computedSatellites,
      imagingWaitResults,
    };
  },
});
</script>

<style scoped>
.simulator {
  max-width: 800px;
  margin: 0 auto;
  padding: 1rem;
}
form {
  border: 1px solid #ccc;
  padding: 1rem;
  margin-bottom: 1rem;
}
.satellite-input,
.aoi-input {
  margin-bottom: 1rem;
}
.results table {
  width: 100%;
  border-collapse: collapse;
}
.results th,
.results td {
  border: 1px solid #ccc;
  padding: 0.5rem;
  text-align: center;
}
.map-wrapper {
  margin-top: 1rem;
}
</style>
