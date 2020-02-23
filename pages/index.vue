<template>
  <div class="wrapper">
    <div class="row no-gutters">
      <div class="col-sm-3">
        <div class="toolbox">
          <div class="menu-wrapper sticky-top bg-white shadow-sm">
            <div class="form-group d-flex">
              <label for="cityName" class="mr-2 col-form-label text-right">縣市</label>
              <div class="flex-fill">
                <select id="cityName" v-model="selectedCounty" class="form-control" @change="onCityChanged">
                  <option v-for="city in cityCounty" :key="city.CityEngName" :value="city.CityName">
                    {{ city.CityName }}
                  </option>
                </select>
              </div>
            </div>
            <div class="form-group d-flex">
              <label for="area" class="mr-2 col-form-label text-right">地區</label>
              <div class="flex-fill">
                <select id="area" v-model="selectedtown" class="form-control" @change="onTownChanged">
                  <option v-for="town in townlist" :key="town.AreaEngName" :value="town.AreaName">
                    {{ town.AreaName }}
                  </option>
                </select>
              </div>
            </div>
            <p class="mb-0 small text-muted text-right">請先選擇區域查看（綠色表示還有口罩）</p>
          </div>
          <ul v-if="storeList.length > 0" class="list-group">
            <template v-for="({properties:store,}, id) in storeList">
              <a :key="store.id" class="list-group-item text-left" @click="togglePopup(id)">
                <h3>{{ store.name }}</h3>
                <p class="mb-0">{{ `成人口罩：${store.mask_adult} | 兒童口罩：${store.mask_child}` }}</p>
                <p class="mb-0">地址：
                  <a :href="`https://www.google.com.tw/maps/place/${store.address}`" target="_blank" title="Google Map">
                    {{ `${store.address}` }}
                  </a>
                </p>
              </a>
            </template>
          </ul>
          <div v-else class="store-list-none">
            <h3>無藥局資料</h3>
          </div>
        </div>
      </div>
      <div class="col-sm-9">
        <div id="map-wrap"></div>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios'
import cityCounty from '@/assets/cityCounty.json'

let osmMap = {}
let markers = []
const icons = {
  green: {},
  gold: {},
  red: {},
  grey: {}
}
export default {
  data () {
    return {
      storeData: [],
      cityCounty,
      selectedCounty: '臺北市',
      selectedtown: '中正區',
      colorBy: 'adult'
    }
  },
  computed: {
    townlist () {
      return this.cityCounty.find(city => city.CityName === this.selectedCounty).AreaList
    },
    storeList () {
      return this.storeData.filter((store) => {
        return store.properties.county === this.selectedCounty && store.properties.town === this.selectedtown
      })
    }
  },
  mounted () {
    const pr = axios.get('https://raw.githubusercontent.com/kiang/pharmacies/master/json/points.json')
      .then((result) => {
        this.storeData = result.data.features
        // center: [25.03, 121.55]

        osmMap = this.$L.map('map-wrap', { // Map Creation
          center: this.storeList[0].geometry.coordinates.reverse(),
          zoom: 15
        })

        this.$L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', { // 做圖磚
          attribution: 'Map data &copy; <a href="https://www.openstreetmap.org/">OpenStreetMap</a> contributors, <a href="https://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>',
          maxZoom: 18
        }).addTo(osmMap)

        this.updateMarker()

        // <h5>${name}</h5>
        // <p>電話：${phone}</p>
        // <p>口罩數量：成人口罩：${mask_adult} | 兒童口罩：${mask_child}</p>
        // <p>地址：<a :href="https://www.google.com.tw/maps/place/${address}" target="_blank" title="Google Map">${address}</a></p>
        // <p>註記：口罩週一至週日 14:30 現場排隊</p>

        return result
      })

    for (const icon in icons) {
      icons[icon] = new this.$L.Icon({
        iconUrl: `https://cdn.rawgit.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-${icon}.png`,
        shadowUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/0.7.7/images/marker-shadow.png',
        iconSize: [25, 41],
        iconAnchor: [12, 41],
        popupAnchor: [1, -34],
        shadowSize: [41, 41]
      })
    }

    return pr
  },
  methods: {
    onCityChanged () {
      this.selectedtown = this.townlist[0].AreaName
      this.onTownChanged()
    },
    onTownChanged () {
      if (this.storeList.length !== 0) {
        osmMap.panTo(this.storeList[0].geometry.coordinates.reverse()) // 移動

        markers.forEach((marker) => { // 把舊 marker 刪掉
          osmMap.removeLayer(marker)
        })
        markers = []

        this.updateMarker()
      }
    },
    updateMarker () {
      this.storeList.forEach((store) => { // 加 marker
        const { name, phone, address, mask_adult: ad, mask_child: ch, note } = store.properties
        const location = store.geometry.coordinates.reverse()
        const maskNum = this.colorBy === 'adult' ? ad : ch
        let marker = {}
        let color = ''

        if (maskNum === 0) {
          color = 'grey'
        } else if (maskNum < 50) {
          color = 'red'
        } else if (maskNum < 100) {
          color = 'gold'
        } else {
          color = 'green'
        }

        marker = this.$L.marker(location, { icon: icons[color] }).addTo(osmMap)
        marker.bindPopup(`<div"><h5>${name}</h5><p>電話：${phone}</p><p>口罩數量：成人口罩：${ad} | 兒童口罩：${ch}</p><p>地址：<a href="https://www.google.com.tw/maps/place/${address}" target="_blank" title="Google Map">${address}</a></p><p>註記：${note}</p></div>`)
        markers.push(marker)
      })
    },
    togglePopup (id) {
      markers[id].togglePopup()
    }
  }
}
</script>

<style lang='scss'>
html {
  font-size: 14px;
}

#map-wrap {
  height: 100vh;
}

.highlight {
  background: #e9ffe3;
}

.menu-wrapper {
  padding: 20px 15px;
}

.store-list-none {
  padding: 0.75rem 1.25rem;
}

.toolbox {
  height: 100vh;
  overflow-y: auto;
  a {
    cursor: pointer;
  }
}

.leaflet-popup-content p {
  margin: 8px 0;
}
</style>
