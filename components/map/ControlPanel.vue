<template>
  <div class="mt-2">
    <div class="px-3 py-2">
      <b-form-checkbox v-model="travel" switch class="text-uppercase" @input="$emit('travel-times-toggled', $event)">Tranvel Times</b-form-checkbox>
      <b-form-checkbox v-model="radius" switch class="text-uppercase" @input="$emit('radius-rings-toggled', $event)">Radius Rings</b-form-checkbox>
      <b-form-group class="mt-2">
        <b-form-radio-group
          v-if="travel"
          v-model="travelMode"
          :options="['walking', 'cycling', 'driving']"
          buttons
          @input="onTravelModeChanged"
          @click="onTravelModeChanged"
          button-variant="outline-primary"
          size="sm"
        ></b-form-radio-group>
      </b-form-group>
      <div class="mt-1">
        <label for="available-sqm" class="mb-0 text-uppercase">Available SQM</label>
        <b-form-input id="available-sqm" v-model="sqm" type="range" min="0" max="30000"></b-form-input>
        <div class="mt-n2 text-center text-warning">0 SQM - 30,000 SQM</div>
      </div>
    </div>
    
    <b-list-group flush class="mt-2">
      <b-list-group-item
        variant="danger"
        button
        @click="$emit('reset-bounds')"
      >
        <div class="py-3">Reset Map Zoom</div>
      </b-list-group-item>
      <b-list-group-item
        v-for="(location, i) in locations"
        :key="location.id"
        button
        variant="primary"
        class="d-flex justify-content-between align-items-center text-light"
        :active="selectedLocationId == location.id"
        @click="onListClick(location)"
      >
        <div>
          <div class="h5">{{ location.name }}</div>
          <small>Total availabiity SQM {{ location.squarefeet }}</small>
        </div>
        <div class="h1">{{ i + 1 }}</div>
      </b-list-group-item>
    </b-list-group>

    <b-card id="minutes" bg-variant="primary" text-variant="white">
      <b-card-text>
        <div>Drive Time In Minutes</div>
        <b-button-group style="width: 200px;">
          <b-button
            v-for="(option, i) in minuteOptions"
            :key="i"
            size="sm"
            :style="`background-color: ${option.variant}; border-color: ${option.variant};`"
            disabled
          >{{ option.text }}</b-button>
        </b-button-group>
      </b-card-text>
    </b-card>
  </div>
</template>

<script>
import { mapState, mapMutations, mapActions } from 'vuex'
import { BIcon, BIconTruck, BIconPiggyBank, BIconBicycle } from 'bootstrap-vue'

export default {
  middleware: 'auth',
  meta: {
    requiresAuth: true
  },
  components: {
    BIcon, BIconTruck, BIconPiggyBank, BIconBicycle
  },
  data: () => ({
    loading: false,
    travel: true,
    radius: true,
    sqm: 2,
    selectedLocationId: null,
    travelMode: null,
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
    }]
  }),
  computed: {
    ...mapState('location', ['locations'])
  },
  async fetch () {
    this.getLocations(this.$route.params.id)
  },
  methods: {
    ...mapActions('location', ['getLocations']),
    onListClick(location) {
      this.selectedLocationId = location.id
      this.$emit('selected', location)
    },
    onTravelModeChanged() {
      this.$nextTick(() => {
        if (this.travelMode)
          this.$emit('iso-input', this.travelMode)
      })
    }
  }
}
</script>
<style scoped>
.list-group-item-danger {
  color: #ffffff;
  background-color: #d14a59;
}
.list-group-item-danger:hover {
  color: #ffffff;
  background-color: #dd3145;
}
#minutes {
  position: absolute;
  z-index: 2;
  left: 340px;
  bottom: 20px;
}
#minutes .card-body {
  padding: 0.6rem;
}
</style>