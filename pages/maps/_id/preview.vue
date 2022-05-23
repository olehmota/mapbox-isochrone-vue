<template>
  <div>
    <b-sidebar visible shadow no-header>
      <div class="py-2">
        <map-control-panel
          @selected="onLocationSelected"
          @reset-bounds="resetBounds"
          @travel-times-toggled="toggleTravelTimes"
          @radius-rings-toggled="toggleRadiusRings"
          @iso-input="getIso"
        />
      </div>
      <div class="control-toggler right-positioned" @click="toggleSidebar"></div>
    </b-sidebar>
    <div id="menu">
      <b-form-radio-group
        v-model="mapOption"
        :options="mapOptions"
        buttons
        @input="onMapModeChange"
        button-variant="outline-primary"
        size="sm"
      ></b-form-radio-group>
    </div>
    <b-card id="details" img-src="https://www.71langridgestreet.com.au/dist/images/building-01.jpg" img-alt="Image" img-top bg-variant="primary" text-variant="white">
      <b-card-text>
        <h3>71 Langridge St</h3>
        <small>71 Langridge St</small>
        <hr>
        <small>Levels available</small>
        <h5>G, 1, 2, 3, 4, 5, 6</h5>
        <small>Net Rent Range($/sqm)</small>
        <h5>POA</h5>
        <small>Owner</small>
        <h5>Domain Hill</h5>
        <small>Leaging agent</small>
        <h5>Lemon Baxter</h5>
      </b-card-text>
    </b-card>
    <div id="map"></div>
  </div>
</template>

<script>
import mapboxgl from '!mapbox-gl'
import * as turf from '@turf/turf'
import 'mapbox-gl/dist/mapbox-gl.css'
import { mapState, mapActions } from 'vuex'
import { BIcon, BIconBicycle, BIconMinecart, BIconOption } from 'bootstrap-vue'

export default {
  layout: 'map',
  components: {
    BIcon, BIconBicycle, BIconMinecart, BIconOption
  },
  data() {
    return {
      map: null,
      showControlbar: true,
      radius: false,
      travel: false,
      mapOption: 'ckpzzf6ee1jyg17qxen56map9',
      mapOptions: [
        { text: 'Street', value: 'ckpzzf6ee1jyg17qxen56map9' },
        { text: 'Satellite', value: 'cl2ktyiap000q15o8zm1ka0ly' },
        { text: 'Dark', value: 'ckd9gicgp06q51iny30d2j9m9' }
      ],
      minuteOptions: [{
        text: '5',
        value: 5,
        variant: '#3c1483'
      }, {
        text: '10',
        value: 10,
        variant: '#532d97'
      }, {
        text: '15',
        value: 15,
        variant: '#9880c3'
      }],
      meters: [
        {
          radius: 1,
          color: '#11b4da'
        }, {
          radius: 3,
          color: '#11b4da'
        }, {
          radius: 5,
          color: '#11b4da'
        }
      ],
      popup: null,
      selectedLocation: null
    }
  },
  computed: {
    ...mapState('location', ['locations']),
    geojson() {
      return {
        type: 'FeatureCollection',
        features: this.locations.map((location, i) => ({
          type: 'Feature',
          properties: {
            index: i + 1,
            name: location.name
          },
          geometry: {
            type: 'Point',
            coordinates: [location.longitude, location.latitude]
          }
        }))
      }
    }
  },
  async mounted() {
    await this.getLocations(this.$route.params.id)
    this.initMap()
  },
  methods: {
    ...mapActions('location', ['getLocations']),
    initMap() {
      if (this.locations.length && !this.selectedLocation)
        this.selectedLocation = this.locations[0]

      mapboxgl.accessToken = 'pk.eyJ1IjoibWFwc3RlciIsImEiOiJjbDJrdWt4ZnIwNm4zM2JzM2xvdTJjZnBnIn0.am-B5SLKHgGwEU1yKY992w'

      let center = [-87.699329, 41.860092]

      if (this.selectedLocation)
        center = [this.selectedLocation.longitude, this.selectedLocation.latitude]

      this.map = new mapboxgl.Map({
        container: 'map',
        style: 'mapbox://styles/mapster/' + this.mapOption,
        center,
        zoom: 18
      })

      this.map.addControl(new mapboxgl.NavigationControl(), 'top-left');

      this.map.on('load', () => {
        // iso layer
        this.minuteOptions.forEach((option) => {
          this.map.addSource(`iso-${option.value}`, {
            type: 'geojson',
            data: {
              'type': 'FeatureCollection',
              'features': []
            }
          })

          this.map.addLayer({
            'id': `iso-layer-${option.value}`,
            'type': 'fill',
            'source': `iso-${option.value}`,
            'layout': {},
            'paint': {
              'fill-color': option.variant,
              'fill-opacity': 0.3
            }
          }, 'poi-label')
        })

        // locations markers
        this.map.addSource('addresses', {
          type: 'geojson',
          data: this.geojson
        })

        this.map.addLayer({
          id: 'addresses-circle',
          type: 'circle',
          source: 'addresses',
          paint: {
            'circle-color': '#11b4da',
            'circle-radius': 16,
            'circle-stroke-width': 2,
            'circle-stroke-color': '#fff'
          }
        });
        this.map.addLayer({
          id: "addresses-label",
          type: "symbol",
          source: "addresses",
          layout: {
            'text-field': ['get', 'index'],
            "text-size": 16
          },
          paint: {
            'text-color': '#ffffff'
          }
        });

        // circles around location
        this.meters.forEach((meter) => {
          const centr = turf.point(center);
          const radius = meter.radius;
          const options = {
            steps: 80,
            units: 'kilometers'
          }
  
          const circle = turf.circle(centr, radius, options);
          this.map.addLayer({
            id: `circle-${meter.radius}`,
            type: 'line',
            source: {
              type: 'geojson',
              data: circle
            },
            paint: {
              'line-color': meter.color,
              'line-opacity': 1,
              'line-width': 2
            }
          })
        })

        // show all locations in the screen
        this.resetBounds()
      })

      this.map.on('click', 'addresses-circle', (e) => {
        this.setLocationSelection(e.features[0])
      });
    },
    showPopUp(coordinates, html) {
      if (this.popup)
        this.popup.remove()

      this.popup = new mapboxgl.Popup()
        .setLngLat(coordinates)
        .setHTML(html)
        .addTo(this.map)
    },
    setLocationSelection(feature) {
      this.showPopUp(feature.geometry.coordinates, feature.properties.name)
      this.showCircles(feature)
      this.map.flyTo({
        center: feature.geometry.coordinates,
        zoom: 12,
        speed: 1.2
      })
    },
    onMapModeChange() {
      this.initMap()
      // this.map.setStyle('mapbox://styles/mapster/' + this.mapOption);
    },
    onLocationSelected(location) {
      this.selectedLocation = location

      this.setLocationSelection({
        type: 'Feature',
        properties: {
          name: location.name
        },
        geometry: {
          type: 'Point',
          coordinates: [location.longitude, location.latitude]
        }
      })
    },
    resetBounds() {
      const bounds = new mapboxgl.LngLatBounds()

      this.geojson.features.forEach((feature) => {
        bounds.extend(feature.geometry.coordinates)
      })

      this.map.fitBounds(bounds, { padding: 80 })
    },
    showCircles(feature) {
      this.meters.forEach((meter) => {
        const centr = turf.point(feature.geometry.coordinates);
        const radius = meter.radius;
        const options = {
          steps: 80,
          units: 'kilometers'
        }
        const circle = turf.circle(centr, radius, options);
        const source = this.map.getSource(`circle-${meter.radius}`)

        if (source) {
          source.setData(circle)
        }
      })
      // const metersToPixelsAtMaxZoom = (meters, latitude) => meters / 0.075 / Math.cos(latitude * Math.PI / 180)
      // const meters = [1000, 3000, 5000]
      // const source = this.map.getSource('point-5000')

      // if (source) {
      //   source.setData({
      //     type: "FeatureCollection",
      //     features: [{
      //       type: "Feature",
      //       properties: {
      //         title: 'aaaaaa'
      //       },
      //       geometry: feature.geometry
      //     }]
      //   })
      // } else {
      //   this.map.addSource('point-5000', {
      //     type: 'geojson',
      //     data: {
      //       type: "FeatureCollection",
      //       features: [{
      //         type: "Feature",
      //         properties: {
      //           title: 'aaaaaa'
      //         },
      //         geometry: feature.geometry
      //       }]
      //     }
      //   })
      // }

      // meters.forEach((meter) => {
      //   const layerId = `circle-${meter}`

      //   if (this.map.getLayer(layerId)) {
      //     this.map.removeLayer(layerId)
      //   }

      //   this.map.addLayer({
      //     id: layerId,
      //     type: 'circle',
      //     source: 'point-5000',
      //     paint: {
      //       'circle-radius': {
      //         stops: [
      //           [0, 0],
      //           [20, metersToPixelsAtMaxZoom(meter, feature.geometry.coordinates[1])]
      //         ],
      //         base: 2
      //       },
      //       'circle-opacity': 0,
      //       'circle-stroke-width': 2,
      //       'circle-stroke-color': '#11b4da'
      //     }
      //   })
      // })
    },
    async getIso(profile) {
      const urlBase = 'https://api.mapbox.com/isochrone/v1/mapbox/'
      const lon = this.selectedLocation.longitude
      const lat = this.selectedLocation.latitude

      this.$axios.get(`${urlBase}${profile}/${lon},${lat}?contours_minutes=5&polygons=true&access_token=${mapboxgl.accessToken}`).then(async (res) => {
        this.map.getSource('iso-5').setData(res.data)
      })
      this.$axios.get(`${urlBase}${profile}/${lon},${lat}?contours_minutes=10&polygons=true&access_token=${mapboxgl.accessToken}`).then(async (res) => {
        this.map.getSource('iso-10').setData(res.data)
      })
      this.$axios.get(`${urlBase}${profile}/${lon},${lat}?contours_minutes=15&polygons=true&access_token=${mapboxgl.accessToken}`).then(async (res) => {
        this.map.getSource('iso-15').setData(res.data)
      })
    },
    toggleSidebar() {
      this.toggleSidebar = !this.toggleSidebar

      if (!this.toggleSidebar) {
        document.getElementsByClassName('b-sidebar')[0].style.left = '-320px'
        document.getElementsByClassName('mapboxgl-ctrl-top-left')[0].style.left = '0'
        document.getElementById('menu').style.left = '46px'
      } else {
        document.getElementsByClassName('b-sidebar')[0].style.left = '0'
        document.getElementsByClassName('mapboxgl-ctrl-top-left')[0].style.left = '328px'
        document.getElementById('menu').style.left = '374px'
      }
    },
    toggleTravelTimes(val) {
      this.map.setLayoutProperty('iso-layer-5', 'visibility', val ? 'visible' : 'none')
      this.map.setLayoutProperty('iso-layer-10', 'visibility', val ? 'visible' : 'none')
      this.map.setLayoutProperty('iso-layer-15', 'visibility', val ? 'visible' : 'none')
    },
    toggleRadiusRings(val) {
      this.map.setLayoutProperty('circle-1', 'visibility', val ? 'visible' : 'none')
      this.map.setLayoutProperty('circle-3', 'visibility', val ? 'visible' : 'none')
      this.map.setLayoutProperty('circle-5', 'visibility', val ? 'visible' : 'none')
    }
  }
}
</script>

<style>
  #map {
    position: absolute;
    top: 0;
    bottom: 0;
    width: 100%;
  }
  .control-toggler {
    position: absolute;
    z-index: 10;
    top: calc(50% - 20px);
    height: 40px;
    width: 8px;
    border-radius: 0;
    border-top-right-radius: 6px;
    border-bottom-right-radius: 6px;
    background: #35495e;
    cursor: pointer;
  }
  .right-positioned {
    right: -8px;
  }
  #menu {
    display: flex;
    position: absolute;
    background: #efefef;
    font-family: 'Open Sans', sans-serif;
    z-index: 1;
    left: 374px;
    top: 10px;
    border-radius: 10px;
  }
  #details {
    display: flex;
    position: absolute;
    width: 300px;
    z-index: 1;
    right: 12px;
    top: 10px;
  }
  #details img {
    height: 200px;
    object-fit: cover;
  }
  .mapboxgl-popup {
    width: 152px;
  }
  .mapboxgl-ctrl-top-left {
    left: 328px;
  }
</style>