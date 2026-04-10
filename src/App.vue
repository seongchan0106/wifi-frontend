<template>
  <div class="page">
    <div class="sidebar">
      <h1>공공 와이파이 지도</h1>

      <div class="search-box">
        <select v-model="sido" @change="onChangeSido">
          <option value="">시도 선택</option>
          <option v-for="item in sidoList" :key="item" :value="item">
            {{ item }}
          </option>
        </select>

        <select v-model="sigungu" :disabled="!sido">
          <option value="">시군구 선택</option>
          <option v-for="item in sigunguList" :key="item" :value="item">
            {{ item }}
          </option>
        </select>

        <input v-model="keyword" placeholder="장소명 / 주소 / SSID 검색" />

        <button @click="searchWifi">검색</button>
        <button @click="resetMap">초기화</button>
        <button @click="toggleHeatmap">
          히트맵  {{ heatOn ? "끄기" : "켜기" }}
        </button>
      </div>

      <p v-if="loading" class="loading-text">검색 중입니다...</p>

      <p class="count">검색 결과: {{ wifiList.length }}건</p>
      
      <div class="stats-box">
        <div class="summary-card total-card">
          <span>전체 와이파이 수</span>
          <strong>{{ summary.total_wifi_count }}건</strong>
        </div>

        <div class="summary-card total-card">
          <span>전체 시도 수</span>
          <strong>{{ summary.total_sido_count }}개</strong>
        </div>

        <div class="summary-card total-card">
          <span>전체 시군구 수</span>
          <strong>{{ summary.total_sigungu_count }}개</strong>
        </div>
      </div>
      
      <div class="summary-box">
        <div class="summary-card">
          <span>검색 결과</span>
          <strong>{{ wifiList.length }}건</strong>
        </div>

        <div class="chart-box">
          <h2>시도별 와이파이 개수</h2>
          <div class="chart-card">
            <canvas id="sidoChart"></canvas>
          </div>
        </div>

        <div class="chart-box">
          <h2>시군구 TOP 10</h2>
          <div class="chart-card">
            <canvas id="sigunguChart"></canvas>
          </div>
        </div>

        <div class="summary-card">
          <span>선택 시도</span>
          <strong>{{ sido || "-" }}</strong>
        </div>

        <div class="summary-card">
          <span>선택 시군구</span>
          <strong>{{ sigungu || "-" }}</strong>
        </div>
      </div>

      <div class="result-list">
        <div
          v-for="wifi in wifiList"
          :key="wifi.id"
          class="result-item"
          @click="selectWifi(wifi)"
        >
          <strong>{{ wifi.place_name || "장소명 없음" }}</strong>
          <p>{{ wifi.sido }} {{ wifi.sigungu }}</p>
          <p>{{ wifi.road_address || "주소 없음" }}</p>
        </div>
      </div>

      <div v-if="selectedWifi" class="detail-box">
        <h2>상세정보</h2>
        <p><strong>장소명:</strong> {{ selectedWifi.place_name || "-" }}</p>
        <p><strong>시도:</strong> {{ selectedWifi.sido || "-" }}</p>
        <p><strong>시군구:</strong> {{ selectedWifi.sigungu || "-" }}</p>
        <p><strong>도로명주소:</strong> {{ selectedWifi.road_address || "-" }}</p>
        <p><strong>SSID:</strong> {{ selectedWifi.wifi_ssid || "-" }}</p>
        <p><strong>위도:</strong> {{ selectedWifi.lat || "-" }}</p>
        <p><strong>경도:</strong> {{ selectedWifi.lng || "-" }}</p>
      </div>
    </div>

    <div id="map"></div>
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";
import L from "leaflet";
import "leaflet/dist/leaflet.css";
import "leaflet.markercluster";
import "leaflet.markercluster/dist/MarkerCluster.css";
import "leaflet.markercluster/dist/MarkerCluster.Default.css";
import Chart from "chart.js/auto";
import "leaflet.heat";

const wifiList = ref([]);
const sidoList = ref([]);
const sigunguList = ref([]);
const loading = ref(false);

const sido = ref("");
const sigungu = ref("");
const keyword = ref("");
const selectedWifi = ref(null);

const summary = ref({
  total_wifi_count: 0,
  total_sido_count: 0,
  total_sigungu_count: 0,
});

const loadSummary = async () => {
  try {
    const response = await fetch("http://127.0.0.1:8000/wifi/stats/summary");
    const data = await response.json();
    summary.value = data;
  } catch (error) {
    alert("통계 정보 불러오기 실패");
  }
};

let map = null;
let markersLayer = null;
let markerMap = new Map();

let sidoChart = null;
let sigunguChart = null;

let heatLayer = null;
const heatOn = ref(false);

onMounted(async () => {
  map = L.map("map").setView([36.5, 127.8], 7);

  L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
    attribution: "&copy; OpenStreetMap contributors",
  }).addTo(map);

  markersLayer = L.markerClusterGroup();
  map.addLayer(markersLayer);

  await loadSummary();
  await loadSidoList();
  await loadSidoChart();
  await loadSigunguChart();
});

const loadSidoList = async () => {
  try {
    const response = await fetch("http://127.0.0.1:8000/sido");
    const data = await response.json();
    sidoList.value = data;
  } catch (error) {
    alert("시도 목록 불러오기 실패");
  }
};

const loadSigunguList = async () => {
  if (!sido.value) {
    sigunguList.value = [];
    sigungu.value = "";
    return;
  }

  try {
    const response = await fetch(
      `http://127.0.0.1:8000/sigungu?sido=${encodeURIComponent(sido.value)}`
    );
    const data = await response.json();
    sigunguList.value = data;
    sigungu.value = "";
  } catch (error) {
    alert("시군구 목록 불러오기 실패");
  }
};

const onChangeSido = async () => {
  await loadSigunguList();
};

const clearMarkers = () => {
  if (markersLayer) {
    markersLayer.clearLayers();
  }
  markerMap.clear();
};

const drawMarkers = (data) => {
  clearMarkers();

  const bounds = [];

  data.forEach((wifi, index) => {
    const lat = Number(wifi.lat);
    const lng = Number(wifi.lng);

    if (!isNaN(lat) && !isNaN(lng)) {
      const marker = L.marker([lat, lng]).bindPopup(`
        <b>${wifi.place_name || "장소명 없음"}</b><br>
        ${wifi.sido || ""} ${wifi.sigungu || ""}<br>
        ${wifi.road_address || ""}<br>
        SSID: ${wifi.wifi_ssid || ""}
      `);

      marker.on("click", () => {
        selectedWifi.value = wifi;
      });

      marker.addTo(markersLayer);
      bounds.push([lat, lng]);

      markerMap.set(index, marker);
    }
  });

  if (bounds.length > 0) {
    map.fitBounds(bounds, { padding: [30, 30] });
  }
};

const drawHeatmap = (data) => {
  if (!map) return;

  if (heatLayer) {
    map.removeLayer(heatLayer);
    heatLayer = null;
  }

  const heatData = data
    .map((wifi) => {
      const lat = Number(wifi.lat);
      const lng = Number(wifi.lng);

      if (isNaN(lat) || isNaN(lng)) return null;
      return [lat, lng, 0.5];
    })
    .filter(Boolean);

  if (heatData.length === 0) {
    console.warn("히트맵 데이터가 없습니다.");
    return;
  }

  if (typeof L.heatLayer !== "function") {
    console.error("L.heatLayer가 로드되지 않았습니다.");
    return;
  }

  heatLayer = L.heatLayer(heatData, {
    radius: 25,
    blur: 15,
    maxZoom: 17,
  });

  if (heatOn.value) {
    heatLayer.addTo(map);
  }
};

const toggleHeatmap = () => {
  heatOn.value = !heatOn.value;

  if (!heatLayer) {
    console.warn("히트맵 레이어가 아직 생성되지 않았습니다.");
    return;
  }

  if (heatOn.value) {
    heatLayer.addTo(map);
  } else {
    map.removeLayer(heatLayer);
  }
};

const searchWifi = async () => {
  try {
    loading.value = true;

    const params = new URLSearchParams();

    if (sido.value) params.append("sido", sido.value);
    if (sigungu.value) params.append("sigungu", sigungu.value);
    if (keyword.value) params.append("keyword", keyword.value);

    const response = await fetch(
      `http://127.0.0.1:8000/wifi/search?${params.toString()}`
    );

    const data = await response.json();
    wifiList.value = data;
    selectedWifi.value = null;

    drawMarkers(data);

    try {
      drawHeatmap(data);
    } catch (heatError) {
      console.error("히트맵 오류:", heatError);
    }

  } catch (error) {
    console.error("검색 오류:", error);
    alert("검색 실패");
  } finally {
    loading.value = false;
  }
};

const selectWifi = (wifi) => {
  selectedWifi.value = wifi;

  const lat = Number(wifi.lat);
  const lng = Number(wifi.lng);

  if (!isNaN(lat) && !isNaN(lng)) {
    map.setView([lat, lng], 16);
  }
};

const resetMap = () => {
  wifiList.value = [];
  sigunguList.value = [];
  selectedWifi.value = null;
  sido.value = "";
  sigungu.value = "";
  keyword.value = "";

  clearMarkers();

  if (heatLayer) {
    map.removeLayer(heatLayer);
    heatLayer = null;
  }

  heatOn.value = false;

  map.setView([36.5, 127.8], 7);
};

const loadSidoChart = async () => {
  try {
    const response = await fetch("http://127.0.0.1:8000/wifi/stats/sido");
    const data = await response.json();

    const labels = data.map((item) => item.sido);
    const counts = data.map((item) => item.count);

    const canvas = document.getElementById("sidoChart");
    if (!canvas) return;

    if (sidoChart) {
      sidoChart.destroy();
    }

    sidoChart = new Chart(canvas, {
      type: "bar",
      data: {
        labels,
        datasets: [
          {
            label: "와이파이 개수",
            data: counts,
          },
        ],
      },
      options: {
        responsive: true,
        maintainAspectRatio: false,
      },
    });
  } catch (error) {
    alert("시도별 차트 불러오기 실패");
  }
};

const loadSigunguChart = async () => {
  try {
    const response = await fetch("http://127.0.0.1:8000/wifi/stats/sigungu-top10");
    const data = await response.json();

    const labels = data.map((item) => item.region);
    const counts = data.map((item) => item.count);

    const canvas = document.getElementById("sigunguChart");
    if (!canvas) return;

    if (sigunguChart) {
      sigunguChart.destroy();
    }

    sigunguChart = new Chart(canvas, {
      type: "bar",
      data: {
        labels,
        datasets: [
          {
            label: "와이파이 개수",
            data: counts,
          },
        ],
      },
      options: {
        indexAxis: "y",
        responsive: true,
        maintainAspectRatio: false,
      },
    });
  } catch (error) {
    alert("시군구 TOP10 차트 불러오기 실패");
  }
};
</script>

<style scoped>
.page {
  display: flex;
  width: 100%;
  height: 100vh;
  overflow: hidden;
}

.loading-text {
  margin-bottom: 10px;
  font-size: 14px;
  color: #2563eb;
  font-weight: bold;
}

.sidebar {
  width: 380px;
  min-width: 380px;
  height: 100vh;
  overflow-y: auto;
  padding: 16px;
  background: #ffffff;
  border-right: 1px solid #ddd;
}

.search-box {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin: 16px 0;
}

.search-box select,
.search-box input,
.search-box button {
  padding: 10px;
  font-size: 14px;
}

.search-box button {
  cursor: pointer;
}

.count {
  margin-bottom: 12px;
  font-weight: bold;
}

.result-list {
  max-height: 320px;
  overflow-y: auto;
  border: 1px solid #eee;
  margin-bottom: 16px;
}

.result-item {
  padding: 12px;
  border-bottom: 1px solid #eee;
  cursor: pointer;
}

.result-item:hover {
  background: #f5f5f5;
}

.result-item p {
  margin: 4px 0;
  font-size: 13px;
}

.detail-box {
  padding: 12px;
  border: 1px solid #ddd;
  background: #fafafa;
}

.detail-box p {
  margin: 8px 0;
  font-size: 14px;
}

#map {
  flex: 1;
  height: 100vh;
}

.summary-box {
  display: grid;
  grid-template-columns: 1fr;
  gap: 8px;
  margin-bottom: 16px;
}

.summary-card {
  padding: 12px;
  border: 1px solid #ddd;
  background: #fafafa;
  border-radius: 8px;
}

.summary-card span {
  display: block;
  font-size: 12px;
  color: #666;
  margin-bottom: 4px;
}

.summary-card strong {
  font-size: 18px;
  font-weight: bold;
}

.stats-box {
  display: grid;
  grid-template-columns: 1fr;
  gap: 8px;
  margin-bottom: 16px;
}

.total-card {
  background: #f0f7ff;
  border: 1px solid #bfdbfe;
}

.chart-box {
  margin-bottom: 16px;
}

.chart-box h2 {
  font-size: 18px;
  margin-bottom: 8px;
}

.chart-card {
  background: #ffffff;
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 12px;
  height: 280px;
}
</style>