<template>
  <div class="page-crous">
    <header class="header-crous">
      <h1>Carte des RU CROUS</h1>
    </header>
    <div class="map-container">
      <div id="map"></div>
      <button class="geoloc-btn" @click="geolocaliser">Me localiser</button>
    </div>
    <div v-if="RUselectionner" class="ru-info">
      <h2>{{ RUselectionner.name || RUselectionner.titre }}</h2>
      <div v-if="RUselectionner.zone"><b>Zone :</b> {{ RUselectionner.zone }}</div>
      <div v-if="RUselectionner.region"><b>Région :</b> {{ RUselectionner.region }}</div>
      <div v-if="RUselectionner.description"><b>Description :</b> {{ RUselectionner.description }}</div>
      <div v-if="RUselectionner.opening"><b>Horaires :</b> {{ RUselectionner.opening }}</div>
      <div v-if="RUselectionner.contact" v-html="RUselectionner.contact"></div>
      <div v-if="RUselectionner.infos" v-html="RUselectionner.infos"></div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'
import 'leaflet.markercluster'
import 'leaflet.markercluster/dist/MarkerCluster.css'
import 'leaflet.markercluster/dist/MarkerCluster.Default.css'

let markerCluster = null
let map        
let markerGeoloc = null
const RUselectionner = ref(null)

const ruIcon = L.icon({
  iconUrl: '/Map-Marker-PNG-File(1).png',
  iconSize: [48, 48],
  iconAnchor: [24, 48],
  popupAnchor: [0, -48],
  shadowUrl: 'https://unpkg.com/leaflet@1.9.4/dist/images/marker-shadow.png',
  shadowSize: [41, 41]
})

const logementIcon = L.icon({
  iconUrl: '/residenceCrous.png',
  iconSize: [36, 36],
  iconAnchor: [18, 36],
  popupAnchor: [0, -36],
  shadowUrl: 'https://unpkg.com/leaflet@1.9.4/dist/images/marker-shadow.png',
  shadowSize: [41, 41]
});

async function recuperationRUVisible(bounds) {
  const sw = bounds.getSouthWest()
  const ne = bounds.getNorthEast()
  const minLat = Math.min(sw.lat, ne.lat)
  const maxLat = Math.max(sw.lat, ne.lat)
  const minLng = Math.min(sw.lng, ne.lng)
  const maxLng = Math.max(sw.lng, ne.lng)
  const url = `http://localhost:1337/api/crousp?filters[latitude][$gt]=${minLat}&filters[latitude][$lt]=${maxLat}&filters[longitude][$gt]=${minLng}&filters[longitude][$lt]=${maxLng}&pagination[pageSize]=1000`
  const response = await fetch(url)
  const data = await response.json()
  return data.data
}

async function recuperationCrous(bounds) {
  const sw = bounds.getSouthWest()
  const ne = bounds.getNorthEast()
  const minLat = Math.min(sw.lat, ne.lat)
  const maxLat = Math.max(sw.lat, ne.lat)
  const minLng = Math.min(sw.lng, ne.lng)
  const maxLng = Math.max(sw.lng, ne.lng)
  const url = `http://localhost:1337/api/logements?filters[latitude][$gt]=${minLat}&filters[latitude][$lt]=${maxLat}&filters[longitude][$gt]=${minLng}&filters[longitude][$lt]=${maxLng}&pagination[pageSize]=1000`
  const response = await fetch(url)
  const data = await response.json()
  return data.data
}

async function afficherClusterUnique(map) {

  if (markerCluster && map.hasLayer(markerCluster)) {
    map.removeLayer(markerCluster)
  }
  markerCluster = L.markerClusterGroup()
  const bounds = map.getBounds()

  const ruData = await recuperationRUVisible(bounds)
  const logementData = await recuperationCrous(bounds)
  const seen = new Set()

  ruData.forEach(resto => {
    const attrs = resto.attributes || resto
    const latitude = attrs.latitude
    const longitude = attrs.longitude
    const name = attrs.name
    const zone = attrs.zone
    const key = `${latitude},${longitude},${name}`

    if (
      latitude !== null &&
      latitude !== undefined &&
      longitude !== null &&
      longitude !== undefined &&
      !seen.has(key)
    ) {
      seen.add(key)
      const marker = L.marker([latitude, longitude], { icon: ruIcon }).bindPopup(`<b>${name}</b><br>${zone || ''}`)
      marker.on('click', () => { RUselectionner.value = attrs })
      markerCluster.addLayer(marker)
    }
  })

  logementData.forEach(lgt => {
    const attrs = lgt.attributes || lgt
    const lat = attrs.latitude
    const lng = attrs.longitude
    const titre = attrs.titre
    const address = attrs.address
    const key = `${lat},${lng},${titre}`

    if (lat && lng && !seen.has(key)) {
      seen.add(key)
      const marker = L.marker([lat, lng], { icon: logementIcon }).bindPopup(`<b>${titre}</b><br>${address || ''}`)
      markerCluster.addLayer(marker)
    }
  })
  map.addLayer(markerCluster)
}

function geolocaliser() {
  navigator.geolocation.getCurrentPosition(
    position => {
      const { latitude, longitude } = position.coords
      map.setView([latitude, longitude], 14)
      if (markerGeoloc) {
        map.removeLayer(markerGeoloc)
      }
      markerGeoloc = L.marker([latitude, longitude], {
        icon: L.icon({
          iconUrl: 'https://unpkg.com/leaflet@1.9.4/dist/images/marker-icon.png',
          iconSize: [25, 41],
          iconAnchor: [12, 41],
          popupAnchor: [1, -34],
          shadowUrl: 'https://unpkg.com/leaflet@1.9.4/dist/images/marker-shadow.png',
          shadowSize: [41, 41]
        })
      }).addTo(map).bindPopup('Vous êtes ici !').openPopup()
    },
  )
}

onMounted(async () => {
  await nextTick()
  map = L.map('map').setView([46.8, 2.5], 6)
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    maxZoom: 18,
    attribution: '© OpenStreetMap contributors'
  }).addTo(map)

  await afficherClusterUnique(map)

  map.on('moveend', async () => {
    await afficherClusterUnique(map)
  })
  map.on('zoomend', async () => {
    await afficherClusterUnique(map)
  })

  geolocaliser()
})
</script>
