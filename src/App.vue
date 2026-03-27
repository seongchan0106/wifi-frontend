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
      </div>

      <p v-if="loading" class="loading-text">검색 중입니다...</p>

      <p class="count">검색 결과: {{ wifiList.length }}건</p>
      
      <div class="summary-box">
        <div class="summary-card">
          <span>검색 결과</span>
          <strong>{{ wifiList.length }}건</strong>
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

const wifiList = ref([]);
const sidoList = ref([]);
const sigunguList = ref([]);
const loading = ref(false);

const sido = ref("");
const sigungu = ref("");
const keyword = ref("");
const selectedWifi = ref(null);

let map = null;
let markersLayer = null;
let markerMap = new Map();

onMounted(async () => {
  map = L.map("map").setView([36.5, 127.8], 7);

  L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
    attribution: "&copy; OpenStreetMap contributors",
  }).addTo(map);

markersLayer = L.markerClusterGroup();
map.addLayer(markersLayer);

  await loadSidoList();
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
  } catch (error) {
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
  map.setView([36.5, 127.8], 7);
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

</style>