<script setup>
import '../assets/leaflet.css'
import { ref, watch, onMounted } from 'vue'
import L from 'leaflet'
import { GeoSearchControl, OpenStreetMapProvider } from 'leaflet-geosearch'
import 'leaflet/dist/leaflet.css'
import postcodeGeoJson from '../assets/postcodes.json'
import companiesGeoJson from '../assets/companies.json'

import * as turf from '@turf/turf'

import Papa from 'papaparse'

const provider = new OpenStreetMapProvider({
  params: {
    countrycodes: 'gb', // Limit search to UK postcodes
  },
})

const searchControl = new GeoSearchControl({
  provider,
  position: 'topleft',
  retainZoomLevel: true,
})

const selectedOption = ref('company')
const areaLookup = ref({})
const companiesData = ref([])
const areaData = ref([])
const showroomData = ref([])

function hashCode(str) {
  let hash = 0
  for (let i = 0; i < str.length; i++) {
    const char = str.charCodeAt(i)
    hash = (hash << 5) - hash + char
    hash |= 0 // Convert to 32bit integer
  }
  return hash
}

function intToRGB(i) {
  const c = (i & 0x00ffffff).toString(16).toUpperCase()

  return '00000'.substring(0, 6 - c.length) + c
}

function getDeliveryColor(day) {
  if (!day) return 'gray'
  switch (day.toLowerCase()) {
    case 'monday':
      return 'green'
    case 'tuesday':
      return 'orange'
    case 'wednesday':
      return 'pink'
    case 'thursday':
      return 'yellow'
    case 'friday':
      return 'blue'
    default:
      return 'gray'
  }
}

function updateCompaniesJson() {
  // TESTING

  // Use Turf.js to merge features with the same name

  const mergedFeatures = {}
  postcodeGeoJson.features.forEach((feature) => {
    const name = feature.properties.company
    if (!mergedFeatures[name]) {
      mergedFeatures[name] = turf.featureCollection([])
    }
    mergedFeatures[name].features.push(feature)
  })
  console.log('Merged features:', mergedFeatures)

  const mergedCollections = Object.keys(mergedFeatures).map((name) => {
    const merged = mergedFeatures[name].features.reduce((acc, feature) => {
      if (!acc) return feature
      return turf.union(turf.featureCollection([acc, feature]), {
        properties: feature.properties,
      })
    }, null)
    merged.properties = { name }
    return merged
  })

  // Convert mergedCollections to a downloadable JSON file
  const mergedJson = JSON.stringify(mergedCollections, null, 2)
  const blob = new Blob([mergedJson], { type: 'application/json' })
  const url = URL.createObjectURL(blob)

  const link = document.createElement('a')
  link.href = url
  link.download = 'merged_postcodes.json'
  link.click()

  URL.revokeObjectURL(url)
}

onMounted(async () => {
  // Create a map instance
  const map = L.map('map').setView([55.5, -3.5], 6)

  L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
    maxZoom: 19,
    attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>',
  }).addTo(map)

  // Load the CSV data for areas TODO: Move to python preload to improve performance
  const areasResponse = await fetch(new URL('../assets/areas.csv', import.meta.url))
  const areasCsvData = await areasResponse.text()
  Papa.parse(areasCsvData, {
    header: true,
    complete: function (results) {
      areaData.value = results.data
    },
  })

  // Create a lookup object for density data
  areaData.value.forEach((row) => {
    areaLookup.value[row.Postcode] = row.Name
  })

  // Load the CSV data for companies
  const companiesResponse = await fetch(new URL('../assets/companies.csv', import.meta.url))
  const companiesCsvData = await companiesResponse.text()
  Papa.parse(companiesCsvData, {
    header: true,
    complete: function (results) {
      companiesData.value = results.data
    },
  })

  // Merge CSV data with companiesGeoJson data
  postcodeGeoJson.features.forEach((feature) => {
    feature.properties.company = areaLookup.value[feature.properties.name]
    const company =
      companiesData.value.find((c) => c.Contact === feature.properties.company) || null
    feature.properties.total = company ? company['TOTAL'] : null
    feature.properties.rep = company ? company['Sales Rep'] : ''
    feature.properties.color = feature.properties.company
      ? `#${intToRGB(hashCode(feature.properties.company))}`
      : '#fff'
    feature.properties.delivery = company ? company['Delivery'] : ''
  })

  // Merge CSV data with companiesGeoJson data
  companiesGeoJson.features.forEach((feature) => {
    const company = companiesData.value.find((c) => c.Contact === feature.properties.name) || null
    feature.properties.total = company ? company['TOTAL'] : null
    feature.properties.rep = company ? company['Sales Rep'] : ''
    feature.properties.color = feature.properties.name
      ? `#${intToRGB(hashCode(feature.properties.name))}`
      : '#fff'
    feature.properties.delivery = company ? company['Delivery'] : ''
  })

  // Create a GeoJSON layer for the postcode data
  const postcodesLayer = L.geoJson(postcodeGeoJson, {
    style,
    onEachFeature,
  })

  // Create a GeoJSON layer for the companies data
  const companiesLayer = L.geoJson(companiesGeoJson, {
    style,
    onEachFeature,
  }).addTo(map)

  // Add a checkbox to toggle postcode visibility
  const postcodeToggle = L.control({ position: 'bottomleft' })

  postcodeToggle.onAdd = function () {
    const div = L.DomUtil.create('div', 'postcode-toggle')
    div.innerHTML = `
      <label>
        <input type="checkbox" id="postcodeCheckbox"/>
        Show Postcodes
      </label>
    `
    return div
  }

  postcodeToggle.addTo(map)

  document.getElementById('postcodeCheckbox').addEventListener('change', (e) => {
    if (e.target.checked) {
      map.removeLayer(companiesLayer)
      map.addLayer(postcodesLayer)
    postcodesLayer.eachLayer((layer) => {
        postcodesLayer.resetStyle(layer);
    });
      showroomLayer.eachLayer((layer) => layer.bringToFront())
    } else {
      map.removeLayer(postcodesLayer)
      map.addLayer(companiesLayer)
      companiesLayer.eachLayer((layer) => {
        companiesLayer.resetStyle(layer);
    });
      showroomLayer.eachLayer((layer) => layer.bringToFront())
    }
  })

  // control that shows state info on hover
  const info = L.control()

  info.onAdd = function (map) {
    this._div = L.DomUtil.create('div', 'info')
    this.update()
    return this._div
  }

  info.update = function (props) {
    const contents = props
      ? [
          props.name ? `<b>${props.name}</b>`: 'Undefined',
          props.company && props.company,
          props.rep && `Rep: ${props.rep}`,
          props.total && `£${props.total}`,
        ]
          .filter((text) => text) // Filter out falsy values like null, undefined, or empty strings
          .join('<br />')
      : 'Hover over a postcode';
    this._div.innerHTML = `<h4>Companies by area</h4>${contents}`;
  }

  info.addTo(map)

  // get color depending on population density value
  function getColor(d) {
    return d > 1000000
      ? '#800026'
      : d > 500000
        ? '#BD0026'
        : d > 200000
          ? '#E31A1C'
          : d > 100000
            ? '#FC4E2A'
            : d > 50000
              ? '#FD8D3C'
              : d > 10000
                ? '#FEB24C'
                : d > 5000
                  ? '#FED976'
                  : d > 0
                    ? '#FFEDA0'
                    : '#999'
  }

  function style(feature) {
    if (selectedOption.value === 'company') {
      return {
        weight: 1,
        opacity: 0.5,
        color: 'white',
        dashArray: '3',
        fillOpacity: 0.4,
        fillColor: feature.properties.color,
      }
    } else if (selectedOption.value === 'delivery') {
      return {
        weight: 1,
        opacity: 0.5,
        color: 'white',
        dashArray: '3',
        fillOpacity: 0.4,
        fillColor: getDeliveryColor(feature.properties.delivery),
      }
    } else {
      return {
        weight: 1,
        opacity: 0.5,
        color: 'white',
        dashArray: '3',
        fillOpacity: 0.7,
        fillColor: getColor(feature.properties.total),
      }
    }
  }

  function highlightFeature(e) {
    const layer = e.target

    layer.setStyle({
      weight: 3,
      color: '#666',
      dashArray: '',
      fillOpacity: 0.7,
    })

    layer.bringToFront()
    showroomLayer.eachLayer((layer) => layer.bringToFront())

    info.update(layer.feature.properties)
  }

  function resetHighlight(e) {
    companiesLayer.resetStyle(e.target)
    showroomLayer.eachLayer((layer) => layer.bringToFront())

    info.update()
  }

  function onEachFeature(feature, layer) {
    layer.on({
      mouseover: highlightFeature,
      mouseout: resetHighlight,
    })
  }

  // ---------------------------------

  const legend = L.control({ position: 'bottomright' })

  function updateLegend() {
    const div = L.DomUtil.create('div', 'info legend')
    let grades,
      labels = []

    if (selectedOption.value === 'all') {
      grades = [0, 5000, 10000, 50000, 100000, 200000, 500000, 1000000]
      for (let i = 0; i < grades.length; i++) {
        const from = grades[i]
        const to = grades[i + 1]
        labels.push(
          `<i style="background:${getColor(from + 1)}"></i> ${from / 1000}${to ? `K &ndash; ${to / 1000}K` : '+'}`,
        )
      }
    } else if (selectedOption.value === 'delivery') {
      labels = ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday'].map(
        (day) => `<i style="background:${getDeliveryColor(day)}"></i> ${day}`,
      )
    }

    div.innerHTML = labels.join('<br>')
    return div
  }

  legend.onAdd = updateLegend
  legend.addTo(map)

  watch(selectedOption, () => {
    legend.remove()
    legend.onAdd = updateLegend
    legend.addTo(map)

    companiesLayer.eachLayer((layer) => {
      companiesLayer.resetStyle(layer)
    })
    
    postcodesLayer.eachLayer((layer) => {
        postcodesLayer.resetStyle(layer);
    });
  })

  // -----------------------------------------

  // Load the CSV data for showrooms
  const showroomsResponse = await fetch(new URL('../assets/showrooms.csv', import.meta.url))
  const showroomsCsvData = await showroomsResponse.text()
  Papa.parse(showroomsCsvData, {
    header: true,
    complete: function (results) {
      showroomData.value = results.data
    },
  })

  // Add showroom points to a new layer
  const showroomLayer = L.layerGroup().addTo(map)

  showroomData.value.forEach((showroom) => {
    if (showroom.Location) {
      const [latitude, longitude] = showroom.Location.split(',').map((coord) =>
        parseFloat(coord.trim()),
      )
      const marker = L.circleMarker([latitude, longitude], {
        radius: 5,
        fillColor: showroom.Company ? `#${intToRGB(hashCode(showroom.Company))}` : 'red',
        color: 'red',
        weight: 2,
        opacity: 1,
        fillOpacity: 0.8,
      }).addTo(showroomLayer)

      marker.bindPopup(
        `<b>Name: ${showroom.Name}</b><br>Company: ${showroom.Company}<br>Address: ${showroom.Address}<br>Products: `,
      )
    }
  })

  // Add a checkbox to toggle showroom visibility
  const showroomToggle = L.control({ position: 'bottomleft' })

  showroomToggle.onAdd = function () {
    const div = L.DomUtil.create('div', 'showroom-toggle')
    div.innerHTML = `
      <label>
        <input type="checkbox" id="showroomCheckbox" checked />
        Show Showrooms
      </label>
    `
    return div
  }

  showroomToggle.addTo(map)

  document.getElementById('showroomCheckbox').addEventListener('change', (e) => {
    if (e.target.checked) {
      map.addLayer(showroomLayer)
    } else {
      map.removeLayer(showroomLayer)
      map.removeLayer(showroomLabelsLayer)
    }
  })

  // Add a dropdown select as part of the bottom left control
  const dropdownControl = L.control({ position: 'bottomleft' })

  dropdownControl.onAdd = function () {
    const div = L.DomUtil.create('div', 'dropdown-control')
    div.innerHTML = `
      <label>
        <select id="dropdownSelect">
          <option value="company">Company</option>
          <option value="delivery">Delivery</option>
        </select>
      </label>
    `
    return div
  }

  dropdownControl.addTo(map)

  document.getElementById('dropdownSelect').addEventListener('change', (e) => {
    selectedOption.value = e.target.value
  })

  // -----------------------------------------
  // Add a layer for showroom name labels
  const showroomLabelsLayer = L.layerGroup()

  showroomData.value.forEach((showroom) => {
    if (showroom.Location) {
      const [latitude, longitude] = showroom.Location.split(',').map((coord) =>
        parseFloat(coord.trim()),
      )
      const backgroundColor = showroom.Company
        ? `#${intToRGB(hashCode(showroom.Company))}`
        : 'white'
      const textColor =
        backgroundColor === 'black' || parseInt(backgroundColor.replace('#', ''), 16) < 0x888888
          ? 'white'
          : 'black'
      const label = L.marker([latitude, longitude], {
        icon: L.divIcon({
          className: 'showroom-label',
          html: `<span style="border: 1px solid ${backgroundColor}; background-color: ${backgroundColor}; color: ${textColor};">${showroom.Name}</span>`,
          iconSize: [0, 0],
        }),
      }).addTo(showroomLabelsLayer)
    }
  })

  map.on('zoomend', () => {
    const showroomLabelsCheckbox = document.getElementById('showroomCheckbox')
    if (map.getZoom() > 9 && showroomLabelsCheckbox.checked) {
      map.addLayer(showroomLabelsLayer)
    } else {
      map.removeLayer(showroomLabelsLayer)
    }
  })

  // Add CSS for showroom labels
  const styleElement = document.createElement('style')
  styleElement.textContent = `
    .showroom-label span {
      font-size: 12px;
      font-weight: bold;
      color: black;
      background-color: white;
      padding: 2px 4px;
      border-radius: 3px;
      box-shadow: 0 0 2px rgba(0, 0, 0, 0.5);
      white-space: nowrap;
    }
  `
  document.head.appendChild(styleElement)

  // -----------------------------------------
  map.addControl(searchControl)
})
</script>

<template>
    <div id="map" ></div>
</template>

<style>
html, body, #map {
    height: 100%;
    margin: 0;
    padding: 0;
}

main {
    height: 100dvh;
    width: 100dvw;
    display: flex;
    justify-content: center;
    align-items: center;
}

#map {
    flex: 1;
}

header {
  line-height: 1.5;
}

.logo {
  display: block;
  margin: 0 auto 2rem;
}

@media (min-width: 1024px) {
  header {
    display: flex;
    place-items: center;
    padding-right: calc(var(--section-gap) / 2);
  }

  .logo {
    margin: 0 2rem 0 0;
  }

  header .wrapper {
    display: flex;
    place-items: flex-start;
    flex-wrap: wrap;
  }
}

.info {
  padding: 6px 8px;
  font:
    14px/16px Arial,
    Helvetica,
    sans-serif;
  background: white;
  background: rgba(255, 255, 255, 0.8);
  box-shadow: 0 0 15px rgba(0, 0, 0, 0.2);
  border-radius: 5px;
}
.info h4 {
  margin: 0 0 5px;
  color: #777;
}
.legend {
  text-align: left;
  line-height: 15px;
  color: #555;
  max-height: 200px;
  overflow-y: auto;
  font-size: 12px;
}
.legend i {
  width: 15px;
  height: 15px;
  float: left;
  margin-right: 8px;
  opacity: 0.7;
}

.main {
  display: flex;
  align-content: center;
  justify-content: center;
  align-items: center;
  height: 100dvh;
}

select {
  padding: 6px 12px;
  font-size: 10px;
  border: 1px solid #ccc;
  border-radius: 4px;
  background-color: #f9f9f9;
  color: #333;
  outline: none;
  transition:
    border-color 0.3s ease,
    box-shadow 0.3s ease;
}

select:focus {
  border-color: #007bff;
  box-shadow: 0 0 5px rgba(0, 123, 255, 0.5);
}

select option {
  padding: 8px;
}
</style>
