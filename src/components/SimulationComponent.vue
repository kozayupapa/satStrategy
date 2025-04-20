<template>
  <div>
    <form @submit.prevent="startSimulation">
      <section>
        <h2 class="text-3xl font-bold text-gray-900">Satellites</h2>
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
              <input type="number" step="any" :value="sat.launchAngle.toFixed(2)" disabled />
            </label>
          </div>
          <div v-else>
            <label>
              Launch Inclination Angle (deg):
              <input type="number" step="any" v-model.lazy.number="sat.launchAngle" />
            </label>
          </div>
          <div>
            <label>
              Launch Location Latitude:
              <input type="number" step="any" v-model.lazy.number="sat.launchLat" />
            </label>
            <label>
              Launch Location Longitude:
              <input type="number" step="any" v-model.lazy.number="sat.launchLon" />
            </label>
          </div>
          <div>
            <label>
              Altitude (km):
              <input type="number" step="any" v-model.lazy.number="sat.altitude" />
            </label>
            <label>
              meanMotion (revs per day):
              <input type="number" step="any" :value="sat.meanMotion?.toFixed(2)" disabled />
            </label>
          </div>
        </div>
        <button
          type="button"
          class="bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-6 rounded-lg shadow-md hover:shadow-lg transition duration-300 text-lg"
          @click="addSatellite"
        >
          Add Satellite
        </button>
      </section>

      <section style="margin-top: 1rem">
        <h2 class="text-3xl font-bold text-gray-900">AOI</h2>
        <div v-for="(aoi, index) in aois" :key="index" style="margin-bottom: 0.5rem">
          <label>
            AOI Latitude:
            <input type="number" step="any" v-model.lazy.number="aoi.lat" />
          </label>
          <label>
            AOI Longitude:
            <input type="number" step="any" v-model.lazy.number="aoi.lon" />
          </label>
        </div>
        <button
          type="button"
          class="bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-6 rounded-lg shadow-md hover:shadow-lg transition duration-300 text-lg"
          @click="addAOI"
        >
          Add AOI
        </button>
      </section>

      <div style="margin-top: 1rem">
        <button
          type="submit"
          class="bg-gradient-to-r from-blue-500 to-purple-600 hover:from-blue-700 hover:to-purple-800 text-white font-bold py-4 px-8 rounded-2xl shadow-lg transform hover:scale-105 transition duration-300"
        >
          🚀 Start Simulation
        </button>
        <button
          type="button"
          class="bg-green-600 hover:bg-green-700 text-white font-bold py-4 px-8 rounded-2xl shadow-lg transform hover:scale-105 transition duration-300"
          @click="optimizeSatellites"
          :disabled="optimizing"
        >
          <svg v-if="optimizing" class="animate-spin h-5 w-5 mr-3" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
            <circle cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4" stroke-linecap="round" fill="none" stroke-opacity="0.25" />
            <path d="M4 12a8 8 0 0 1 8-8" stroke="currentColor" stroke-width="4" stroke-linecap="round" fill="none" />
          </svg>
          {{ optimizing ? "🔍 Optimizing..." : "🔍 Optimize Satellites" }}
        </button>
      </div>
    </form>
    <div v-if="simulationStarted" style="margin-top: 2rem">
      <!-- マップコンポーネントへ、計算済みの各衛星軌道データと AOI 情報を渡す -->
      <MapComponent :access-token="accessToken" :satellites="computedSatellites" :aois="aois" />
    </div>
    <!-- 撮像待ち時間の結果表示 -->
    <div v-if="imagingWaitResults.length > 0" class="results">
      <h2 class="text-3xl font-bold text-gray-900">Imaging Wait Times</h2>
      <table>
        <thead>
          <tr>
            <th>Satellite Index</th>
            <th>AOI Index</th>
            <th>Average Wait Time (hour)</th>
            <th>Maximum Wait Time (hour)</th>
            <th>Imaging Times (hour)</th>
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
    <div v-if="aggregatedImagingWaitResults.length > 0" class="results">
      <h2 class="text-3xl font-bold text-gray-900">Aggregated Imaging Wait Times</h2>
      <table>
        <thead>
          <tr>
            <th>AOI Index</th>
            <th>Average Wait Time (hour)</th>
            <th>Maximum Wait Time (hour)</th>
            <th>Imaging Times (hour)</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="result in aggregatedImagingWaitResults" :key="result.aoiIndex">
            <td>{{ result.aoiIndex }}</td>
            <td>{{ result.avgWait !== null ? result.avgWait.toFixed(2) : "N/A" }}</td>
            <td>{{ result.maxWait !== null ? result.maxWait.toFixed(2) : "N/A" }}</td>
            <td>{{ result.imagingTimes.join(", ") }}</td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, computed } from "vue";
import MapComponent from "./MapComponent.vue";
import * as satellite from "satellite.js";
import { RandomForestRegression } from "ml-random-forest";

const R_EARTH = 6378.137; // 地球半径 [km]
const MU = 398600.4418; // 地球の重力定数 [km^3/s^2]
export const TIME_SCALE = 10; // シミュレーション時刻と実時間の比 1 step = 10 sec これをある程度の小さく解像度が足りず近距離が判定できない
export const SIM_DAYS = 10;
export const SIM_DURATION = (60 * 60 * 24 * SIM_DAYS) / TIME_SCALE;

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
  meanMotion: number | null; // revs per day
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
        launchLat: 33.61,
        launchLon: 142.83,
        altitude: 500, // km
        meanMotion: null,
      },
      {
        orbitType: ORBIT_TYPES.INCLINED,
        launchAngle: 45, // sun-synchronous では固定
        launchLat: 33.61,
        launchLon: 142.83,
        altitude: 550, // km
        meanMotion: null,
      },
    ]);

    // ここで、satellitesRef をグローバルな reactive 変数として定義\
    /*
    const satellitesRef = ref<SatelliteInput[]>([
      { orbitType: 'inclined', launchLat: 30, launchLon: 140, altitude: 500, launchAngle: 97.8 }
    ]);*/

    const aois = ref<AOI[]>([
      { lat: 35.77, lon: 139.82 }, //Tokyo
      { lat: 24.33, lon: 119.78 }, //Taiwan
      { lat: 50.45, lon: 30.52 }, //Ukraine
      { lat: 31.58, lon: 34.98 }, //Israel
      { lat: 29.77, lon: -102.45 }, //USA-Mexico
    ]);

    const simulationStarted = ref(false);

    const addSatellite = () => {
      satellitesRef.value.push({
        orbitType: ORBIT_TYPES.SUN_SYNCHRONOUS,
        launchAngle: 97.8,
        launchLat: 35.61,
        launchLon: 141.83,
        altitude: 500,
        meanMotion: null,
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
      const dLat = toRadians(lat2 - lat1);
      const dLon = toRadians(lon2 - lon1);
      const a = Math.sin(dLat / 2) ** 2 + Math.cos(toRadians(lat1)) * Math.cos(toRadians(lat2)) * Math.sin(dLon / 2) ** 2;
      const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
      return R_EARTH * c;
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

    const J2 = 1.08262668e-3;
    // 太陽同期軌道に必要な昇交点回帰率（rad/s）
    // 1年 = 365.2422日 → 365.2422 * 86400秒
    const dOmegadt_required = (-2 * Math.PI) / (365.2422 * 86400); // 約 -1.99e-7 rad/s

    /**
     * 与えられた高度 (km) に対して必要な太陽同期軌道の傾斜角（deg）を計算する関数
     * 円軌道 (ecc=0) を仮定
     */
    const calculateSunSyncInclination = (altitude: number): number => {
      // 軌道半径
      const a = R_EARTH + altitude; // km
      // 平均運動 (rad/s)
      const n = Math.sqrt(MU / Math.pow(a, 3));
      // cos i を計算
      // dOmegadt_required = - (3/2) * J2 * (RE/a)^2 * n * cos i
      // → cos i = - (dOmegadt_required) / [ (3/2) * J2 * (RE/a)^2 * n ]
      const cos_i = -dOmegadt_required / ((3 / 2) * J2 * Math.pow(R_EARTH / a, 2) * n);
      // 防衛的に cos_i を [-1,1] にクランプ
      const clampedCosI = Math.max(-1, Math.min(1, cos_i));
      // i (rad) を計算
      const i_rad = Math.acos(clampedCosI);
      // 太陽同期軌道は後退軌道 (i > 90°) となるため、
      // 必要な傾斜角は 180° - i (deg)
      const i_deg = toDegrees(i_rad);
      return 180 - i_deg;
    };

    /**
     * ユーザー入力からダミーTLEを生成する関数
     * @param sat ユーザーが入力した衛星パラメータ
     * @returns { line1: string, line2: string } ダミーTLEの2行
     */
    const generateDummyTLE = (sat: SatelliteInput): { line1: string; line2: string } => {
      // 'sun-synchronous' の場合は打ち上げ傾斜角を 97.8 固定
      const inclination = sat.orbitType === ORBIT_TYPES.SUN_SYNCHRONOUS ? calculateSunSyncInclination(sat.altitude) : sat.launchAngle;
      if (sat.orbitType === ORBIT_TYPES.SUN_SYNCHRONOUS) sat.launchAngle = inclination;
      // RAAN を打ち上げ経度として簡易設定
      const raan = sat.launchLon;
      //const eccentricity = 0;       // 円軌道と仮定
      const argPerigee = 0; // 近地点引数：0
      const meanAnomaly = 0; // 平均近点角：0

      // 平均運動 (revs per day) の計算
      const semiMajorAxis = R_EARTH + sat.altitude; // km
      const n = Math.sqrt(MU / Math.pow(semiMajorAxis, 3)); // rad/s
      const meanMotion = (n * 86400) / (2 * Math.PI); // revs per day
      sat.meanMotion = meanMotion;
      console.log(`alt:${sat.altitude},maj:${semiMajorAxis},n:${n},mot:${meanMotion}`);

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

    const lateralTolerance = 9; // 真横条件の許容誤差（度）
    const offNadirMin = 15; // Off-Nadir 角の下限 (度)
    const offNadirMax = 50; // Off-Nadir 角の上限 (度)
    const TIME_SCALE_LOCAL = TIME_SCALE;

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
              // console.log(
              //   `S${satIndex}A${aoiIndex}[${i}] lat${currentPos.lat.toFixed(2)}lon${currentPos.lng.toFixed(2)} Distance: ${distance.toFixed(2)} km, offNadirDeg: ${offNadirDeg.toFixed(2)}`,
              // );
              const last = imagingTimes.at(-1);
              const current = Math.round((i * TIME_SCALE_LOCAL) / 2 / 36) / 100;
              if (offNadirDeg >= offNadirMin && offNadirDeg <= offNadirMax && (last ? current - last > 0.2 : true)) {
                imagingTimes.push(current);
              }
            }
          }
          if (imagingTimes.length >= 2) {
            const intervals: number[] = [];
            intervals.push(imagingTimes[0]);
            for (let j = 0; j < imagingTimes.length - 1; j++) {
              intervals.push(imagingTimes[j + 1] - imagingTimes[j]);
            }
            const avgInterval = intervals.reduce((sum, dt) => sum + dt, 0) / intervals.length;
            const avgWait = Math.round(avgInterval * 100) / 100;
            const maxWait = Math.round(Math.max(...intervals) * 100) / 100;
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

    const aggregatedImagingWaitResults = computed(() => {
      const results: Record<number, { avgWait: number; maxWait: number; count: number; imagingTimes: number[] }> = {};

      imagingWaitResults.value.forEach(({ aoiIndex, imagingTimes }) => {
        if (!results[aoiIndex]) {
          results[aoiIndex] = { avgWait: 0, maxWait: 0, count: 0, imagingTimes: [] };
        }
        results[aoiIndex].imagingTimes.push(...imagingTimes);
      });

      return Object.entries(results).map(([aoiIndex, data]) => {
        if (data.imagingTimes.length >= 2) {
          data.imagingTimes.sort((a, b) => a - b);
          const intervals: number[] = [];
          intervals.push(data.imagingTimes[0]);
          for (let j = 0; j < data.imagingTimes.length - 1; j++) {
            intervals.push(data.imagingTimes[j + 1] - data.imagingTimes[j]);
          }
          const avgInterval = intervals.reduce((sum, dt) => sum + dt, 0) / intervals.length;
          const avgWait = Math.round(avgInterval * 100) / 100;
          const maxWait = Math.round(Math.max(...intervals) * 100) / 100;
          return {
            aoiIndex: Number(aoiIndex),
            avgWait,
            maxWait,
            imagingTimes: data.imagingTimes,
          };
        }
        return {
          aoiIndex: Number(aoiIndex),
          avgWait: 0,
          maxWait: 0,
          imagingTimes: data.imagingTimes,
        };
      });
    });
    const startSimulation = () => {
      simulationStarted.value = true;
    };
    const optimizing = ref(false);
    // 衛星の軌道データを生成する（computedSatellites と同様の処理）
    const computeSatellitesForCandidate = (candidate: SatelliteInput[]): SatelliteOrbit[] => {
      return candidate.map((satInput) => {
        const { line1, line2 } = generateDummyTLE(satInput);
        const satrec = satellite.twoline2satrec(line1, line2);
        const orbitData: Array<{ lat: number; lng: number }> = [];
        const startTime = new Date();
        for (let t_sim = 0; t_sim <= SIM_DURATION; t_sim++) {
          const t_offset_sec = t_sim * TIME_SCALE;
          const currentTime = new Date(startTime.getTime() + t_offset_sec * 1000);
          const posVel = satellite.propagate(satrec, currentTime);
          if (posVel.position && typeof posVel.position !== "boolean") {
            const gmst = satellite.gstime(currentTime);
            const geodetic = satellite.eciToGeodetic(posVel.position, gmst);
            const lat = satellite.degreesLat(geodetic.latitude);
            const lon = satellite.degreesLong(geodetic.longitude);
            orbitData.push({ lat, lng: lon });
          }
        }
        return { orbitData };
      });
    };

    // -----------------------
    // ② 新たな heuristicCost 関数
    // 各候補に対して、シミュレーションで得られた各 AOI の maxWait の合計をコストとする
    // -----------------------
    const heuristicCost = (candidate: SatelliteInput[]): number => {
      // 候補パラメータに基づく衛星軌道データをシミュレーション
      const satOrbits = computeSatellitesForCandidate(candidate);

      // AOI ごとに各衛星からの撮像時刻を集約する
      const aggregated: Record<number, number[]> = {};
      satOrbits.forEach((satOrbit, satIndex) => {
        aois.value.forEach((aoi, aoiIndex) => {
          const imagingTimes: number[] = [];
          for (let i = 0; i < satOrbit.orbitData.length - 1; i++) {
            const currentPos = satOrbit.orbitData[i];
            const nextPos = satOrbit.orbitData[i + 1];
            const heading = computeBearing(currentPos.lat, currentPos.lng, nextPos.lat, nextPos.lng);
            const lateral1 = (heading + 90) % 360;
            const lateral2 = (heading + 270) % 360;
            const bearingToAOI = computeBearing(currentPos.lat, currentPos.lng, aoi.lat, aoi.lon);
            const diff1 = angleDifference(bearingToAOI, lateral1);
            const diff2 = angleDifference(bearingToAOI, lateral2);
            if (diff1 <= lateralTolerance || diff2 <= lateralTolerance) {
              // Off-Nadir 角の計算
              const distance = haversineDistance(currentPos.lat, currentPos.lng, aoi.lat, aoi.lon);
              // offNadir = arctan(distance / altitude)
              const offNadirDeg = toDegrees(Math.atan(distance / candidate[satIndex].altitude));
              const last = imagingTimes.length ? imagingTimes[imagingTimes.length - 1] : null;
              const current = Math.round((i * TIME_SCALE) / 2 / 36) / 100;
              if (offNadirDeg >= offNadirMin && offNadirDeg <= offNadirMax && (last ? current - last > 0.2 : true)) {
                imagingTimes.push(current);
              }
            }
          }
          // AOI ごとに撮像時刻を集約
          if (!aggregated[aoiIndex]) {
            aggregated[aoiIndex] = [];
          }
          aggregated[aoiIndex].push(...imagingTimes);
        });
      });

      // 各 AOI について、撮像時刻からインターバルを計算し、maxWait を算出
      let totalCost = 0;
      Object.keys(aggregated).forEach((aoiIndexStr) => {
        const times = aggregated[parseInt(aoiIndexStr, 10)];
        if (times.length >= 2) {
          times.sort((a, b) => a - b);
          const intervals: number[] = [];
          intervals.push(times[0]); // 初回撮像時刻を初期待ち時間とする
          for (let j = 0; j < times.length - 1; j++) {
            intervals.push(times[j + 1] - times[j]);
          }
          const maxWait = Math.round(Math.max(...intervals) * 100) / 100;
          totalCost += maxWait;
        } else {
          // 撮像時刻が不足している場合は高いペナルティを課す
          totalCost += 100;
        }
      });

      return totalCost;
    };

    // -----------------------
    // ③ 候補の特徴量をフラット化する関数
    // 各衛星について、launchLat, launchLon, altitude, launchAngle の4要素を順に並べる
    // -----------------------
    const flattenCandidate = (candidate: SatelliteInput[]): number[] => {
      return candidate.flatMap((sat) => [sat.launchLat, sat.launchLon, sat.altitude, sat.launchAngle]);
    };

    // -----------------------
    // ④ 最適化処理内で、RandomForest を利用してサロゲート最適化する例
    // -----------------------
    const optimizeSatellites = async () => {
      optimizing.value = true;
      // 疑似的な待機（実際の処理時間に応じて調整）
      await new Promise((resolve) => {
        setTimeout(resolve, 100);
      });

      // (1) 学習用データ生成：候補パラメータ群を多数生成して、各候補の実際コストを計算
      /*
      const numSamples = 50;
      const trainingSet: number[][] = [];
      const trainingLabels: number[] = [];
      for (let i = 0; i < numSamples; i++) {
        const candidate = satellitesRef.value.map((sat) => {
          const newSat = { ...sat };
          newSat.launchLat = sat.launchLat + (Math.random() - 0.5) * 45;
          newSat.launchLon = sat.launchLon + (Math.random() - 0.5) * 180;
          newSat.altitude = sat.altitude + (Math.random() - 0.5) * 200;
          if (sat.orbitType === ORBIT_TYPES.INCLINED) {
            newSat.launchAngle = sat.launchAngle + (Math.random() - 0.5) * 20;
          } else {
            newSat.launchAngle = calculateSunSyncInclination(newSat.altitude);
          }
          return newSat;
        });
        const features = flattenCandidate(candidate);
        const cost = heuristicCost(candidate);
        trainingSet.push(features);
        trainingLabels.push(cost);
      }

      // (2) RandomForestRegression の学習（npm install ml-random-forest でインストール済み）
      const rfOptions = {
        seed: 3,
        maxFeatures: 0.8,
        replacement: true,
        nEstimators: 25,
      };
      const rf = new RandomForestRegression(rfOptions);
      rf.train(trainingSet, trainingLabels);
      */

      // (3) サロゲートモデルを用いて新たな候補を生成し、予測コストが最小となる候補を選択
      const numPredictionCandidates = 100;
      let bestPredictedCost = Infinity;
      let bestCandidate: SatelliteInput[] = satellitesRef.value.map((sat) => ({ ...sat }));
      for (let i = 0; i < numPredictionCandidates; i++) {
        const candidate = satellitesRef.value.map((sat) => {
          const newSat = { ...sat };
          newSat.launchLat = sat.launchLat + (Math.random() - 0.5) * 45;
          newSat.launchLon = sat.launchLon + (Math.random() - 0.5) * 180;
          newSat.altitude = sat.altitude + (Math.random() - 0.5) * 200;
          if (sat.orbitType === ORBIT_TYPES.INCLINED) {
            newSat.launchAngle = sat.launchAngle + (Math.random() - 0.5) * 30;
          } else {
            newSat.launchAngle = calculateSunSyncInclination(newSat.altitude);
          }
          return newSat;
        });
        /*
        const features = flattenCandidate(candidate);
        const [predictedCost] = rf.predict([features]); // 配列の最初の要素を取得
        */
        const predictedCost = heuristicCost(candidate);

        if (predictedCost < bestPredictedCost) {
          bestPredictedCost = predictedCost;
          bestCandidate = candidate.map((sat) => ({ ...sat }));
        }
      }

      // (4) 最適候補を実際のヒューリスティック評価で確認
      const trueCost = heuristicCost(bestCandidate);
      console.log("Best candidate true cost:", trueCost);

      // (5) 最適候補を反映
      satellitesRef.value = bestCandidate;

      optimizing.value = false;
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
      aggregatedImagingWaitResults,
      optimizing,
      optimizeSatellites,
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
