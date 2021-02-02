<template>
  <div class="home">
    <div class="row no-gutters">
      <div class="col-3 border rounded d-lg-block d-none">
        <div class="sideBar px-3">
          <div class="sticky-top bg-white">
            <div class="form-group">
              <label for="county">縣市</label>
              <select
                class="form-control"
                id="county"
                v-model="select.county"
                @change="select.town=''"
              >
                <option value="請選擇縣市">請選擇縣市</option>
                <option v-for="county in countyNames" :key="county.CityName">
                  {{ county.CityName }}
                </option>
              </select>
            </div>
            <div class="form-group">
              <label for="town">地區</label>
              <select
                class="form-control"
                id="town"
                v-if="select.county.length"
                v-model="select.town"
                @change="updateMap"
              >
                <option value="請選擇地區">請選擇地區</option>
                <option
                  :value="town.AreaName"
                  v-for="town in countyNames.find(
                    (item) => item.CityName === select.county
                  ).AreaList"
                  :key="town.AreaName"
                >
                  {{ town.AreaName }}
                </option>
              </select>
            </div>
            <hr class="row" />
          </div>
          <div>
            <template v-for="(item, key) in data">
              <a
                href="#"
                :key="key"
                v-if="
                  item.properties.county === select.county &&
                  item.properties.town === select.town
                "
                class="rounded-lg d-block p-4 mb-3"
                :class="{
                  green:
                    item.properties.mask_adult || item.properties.mask_child,
                  gray:
                    item.properties.mask_adult === 0 &&
                    item.properties.mask_child === 0,
                }"
                @click="panTo(item)"
              >
                <h3 class="text-dark">{{ item.properties.name }}</h3>
                <p class="text-dark">電話: {{ item.properties.phone }}</p>
                <p class="text-dark">地址: {{ item.properties.address }}</p>
                <p class="d-flex justify-content-between mb-0">
                  <span class="text-light bg-danger px-3 py-2 rounded-lg">
                    成人口罩: {{ item.properties.mask_adult }}
                  </span>
                  <span class="text-light bg-warning px-3 py-2 rounded-lg">
                    兒童口罩: {{ item.properties.mask_child }}
                  </span>
                </p>
              </a>
            </template>
          </div>
        </div>
      </div>
      <div class="col-lg-9 col-12">
        <div id="map"></div>
      </div>
      <div class="padBar d-block d-lg-none position-absolute bg-white w-100 position-relative">
        <button class="arrowDown position-absolute right-0 btn">
          <i class="fas fa-chevron-down text-md"></i>
        </button>
        <button class="arrowOn position-absolute right-0 btn d-none">
          <i class="fas fa-chevron-up"></i>
        </button>
        <div class="row mt-4 px-3">
          <div class="form-group col-6">
            <label for="county" class="text-nowrap mr-2 mb-0">縣市</label>
            <select
              class="form-control"
              id="county"
              v-model="select.county"
              @change="select.town=''"
            >
              <option value="請選擇縣市">請選擇縣市</option>
              <option v-for="county in countyNames" :key="county.CityName">
                {{ county.CityName }}
              </option>
            </select>
          </div>
          <div class="form-group col-6">
            <label for="town" class="text-nowrap mr-2 mb-0">地區</label>
            <select
              class="form-control"
              id="town"
              v-model="select.town"
              v-if="select.county.length"
              @change="updateMap()"
            >
              <option value="請選擇地區">請選擇地區</option>
              <option
                :value="town.AreaName"
                v-for="town in countyNames.find(
                  (item) => item.CityName === select.county
                ).AreaList"
                :key="town.AreaName"
              >
                {{ town.AreaName }}
              </option>
            </select>
          </div>
          <pad-search :result="data" :choose="select" @updatePlace="panTo"></pad-search>
        </div>
      </div>
    </div>
  </div>
</template>
<style lang="scss">
*,
*::before,
*::after {
  box-sizing: border-box;
}
body {
  position: relative;
  overflow: hidden;
}
#map {
  height: 100vh;
  width: 100%;
}
a:hover {
  text-decoration: none;
}
.sideBar {
  overflow-y: auto;
  height: 100vh;
}
.green {
  background-color: #05ff65;
}
.gray {
  background-color: #d0d0d0;
}
.rounded-lg {
  border-radius: 14px;
}
.padBar {
  bottom: 0;
  height: 300px;
  z-index: 1000;
}
.slideOn{
  animation-name: displayPadBarSlide;
  animation-duration: 3s;
  animation-fill-mode: forwards;
}
.slideDown{
  animation-name: hidePadBarSlide;
  animation-duration: 3s;
  animation-fill-mode: forwards;
}
.right-0{
  right: 0;
}
.pointer{
  cursor: pointer;
}
@keyframes displayPadBarSlide {
  0%   {bottom:-200px;}
  100% {bottom:0;}
}
@keyframes hidePadBarSlide {
  0%   {bottom:0;}
  100% {bottom:-200px;}
}
</style>
<script>
import L from 'leaflet';
import countyNames from '@/assets/city.json';
import $ from 'jquery';
import PadSearch from '../components/PadSearchResult.vue';

let osmMap = {};

const iconSet = {
  iconSize: [25, 40],
};
const myIcon = {
  green: new L.Icon({
    iconUrl: 'https://cdn.rawgit.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-green.png', ...iconSet,
  }),
  gray: new L.Icon({
    iconUrl: 'https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-grey.png', ...iconSet,
  }),
};

export default {
  components: { PadSearch },
  data() {
    return {
      data: [],
      select: {
        county: '臺北市',
        town: '內湖區',
      },
      countyNames,
    };
  },
  created() {
  },
  methods: {
    updateMap() {
      this.removeMarks();
      const pharmacies = this.data.filter((item) => {
        if (!this.select.town) {
          return item.properties.county === this.select.county;
        }
        return item.properties.town === this.select.town;
      });
      pharmacies.forEach((item) => {
        const { properties, geometry } = item;
        const icon = properties.mask_adult
        || properties.mask_child ? myIcon.green : myIcon.gray;
        L.marker([geometry.coordinates[1], geometry.coordinates[0]],
          { icon }).addTo(osmMap).bindPopup(`<h5 class="mb-1">藥局名稱: ${properties.name}</h5><br><p class="mt-0 mb-0 text-sm">電話: ${properties.phone}</p><br><p class="mt-0 mb-0 text-sm">成人口罩: ${properties.mask_adult}</p><br><p class="mt-0 mb-0 text-sm">兒童口罩: ${properties.mask_child}</p>`);
      });
      this.panTo(pharmacies[0]);
    },
    removeMarks() {
      osmMap.eachLayer((layer) => {
        if (layer instanceof L.Marker) {
          osmMap.removeLayer(layer);
        }
      });
    },
    panTo(item) {
      const { properties, geometry } = item;
      const icon = properties.mask_adult || properties.mask_child ? myIcon.green : myIcon.grey;
      osmMap.panTo([geometry.coordinates[1], geometry.coordinates[0]], icon);
      L.marker([geometry.coordinates[1], geometry.coordinates[0]],
        { icon }).addTo(osmMap).bindPopup(`<h5 class="mb-1">藥局名稱: ${properties.name}</h5><br><p class="mt-0 mb-0 text-sm">電話: ${properties.phone}</p><br><p class="mt-0 mb-0 text-sm">成人口罩: ${properties.mask_adult}</p><br><p class="mt-0 mb-0 text-sm">兒童口罩: ${properties.mask_child}</p>`).openPopup();
    },
  },
  mounted() {
    osmMap = L.map('map', {
      center: [25.03, 121.55],
      zoom: 15,
    });
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
    }).addTo(osmMap);

    const url = 'https://raw.githubusercontent.com/kiang/pharmacies/master/json/points.json';
    this.axios.get(url).then((res) => {
      this.data = res.data.features;
      this.updateMap();
    });
    $('.arrowDown').click(() => {
      $('.padBar').addClass('slideDown').removeClass('slideOn');
      $('.arrowDown').addClass('d-none');
      $('.arrowOn').removeClass('d-none');
    });
    $('.arrowOn').click(() => {
      $('.padBar').addClass('slideOn').removeClass('slideDown');
      $('.arrowOn').addClass('d-none');
      $('.arrowDown').removeClass('d-none');
    });
  },
};
</script>
