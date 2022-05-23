<template>
  <validation-observer ref="form" v-slot="{ handleSubmit }">
    <b-form @submit.prevent="handleSubmit(onSubmit)">
      <validation-provider
        name="Name"
        rules="required"
        v-slot="validationContext"
      >
        <b-form-group label="Name:" label-for="name">
          <b-form-input
            id="name"
            v-model="settings.name"
            placeholder="Map name"
            @input="CLEAR_ERROR"
            :state="getValidationState(validationContext)"
          />
          <b-form-invalid-feedback>
            {{ validationContext.errors[0] }}
          </b-form-invalid-feedback>
        </b-form-group>
      </validation-provider>

      <validation-provider
        name="Description"
        rules="required"
        v-slot="validationContext"
      >
        <b-form-group label="Description:" label-for="description">
          <b-form-textarea
            id="description"
            v-model="settings.description"
            placeholder="Description"
            @input="CLEAR_ERROR"
            :state="getValidationState(validationContext)"
          />
          <b-form-invalid-feedback>
            {{ validationContext.errors[0] }}
          </b-form-invalid-feedback>
        </b-form-group>
      </validation-provider>

      <b-form-group label="SEO keywords:" label-for="seo">
        <b-form-tags
          input-id="seo"
          v-model="settings.seo"
          placeholder="SEO keywords"
        />
      </b-form-group>

      <b-form-group label="URL(Copy)" label-for="url">
        <b-input-group>
          <b-form-input id="url" v-model="settings.url" readonly />
          <b-input-group-append>
            <b-button><b-icon-files></b-icon-files></b-button>
          </b-input-group-append>
        </b-input-group>
      </b-form-group>

      <validation-provider
        name="Map Template"
        rules="required"
        v-slot="validationContext"
      >
        <b-form-group label="Map Template:" label-for="template">
          <b-form-select
            id="template"
            v-model="settings.templateId"
            :options="templates"
            value-field="id"
            text-field="name"
            @input="CLEAR_ERROR"
            :state="getValidationState(validationContext)"
          >
            <template #first>
              <b-form-select-option :value="null" disabled>-- Please select a template --</b-form-select-option>
            </template>
          </b-form-select>
          <b-form-invalid-feedback>
            {{ validationContext.errors[0] }}
          </b-form-invalid-feedback>
        </b-form-group>
      </validation-provider>
      
      <b-form-group label="Drivetime Rings">
        <div v-for="(drivetimeRing, i) in settings.drivetimeRings" :key="i" class="d-flex ml-4 mt-1 align-items-center">
          <label style="width: 60px;">{{ drivetimeRing.text }}</label>
          <b-form-radio-group
            v-model="drivetimeRing.on"
            :options="[{ text: 'On', value: true }, { text: 'Off', value: false }]"
            buttons
            button-variant="outline-primary"
            class="ml-2"
          ></b-form-radio-group>
        </div>
      </b-form-group>

      <b-form-group label="Distance Rings">
        <div v-for="(distanceRing, i) in settings.distanceRings" :key="i" class="d-flex ml-4 mt-1">
          <b-input-group append="miles" style="width: 200px;">
            <b-form-input v-model="distanceRing.miles"></b-form-input>
          </b-input-group>
          <b-form-radio-group
            v-model="distanceRing.on"
            :options="[{ text: 'On', value: true }, { text: 'Off', value: false }]"
            buttons
            button-variant="outline-primary"
            class="ml-2"
          ></b-form-radio-group>
        </div>
      </b-form-group>
      
      <b-form-group
        label="Basemap Selector:"
        label-for="basemap-selector"
        class="d-flex align-items-center"
        label-class="mr-2 mb-1"
      >
        <b-form-checkbox id="basemap-selector" v-model="settings.basemapSelector" switch></b-form-checkbox>
      </b-form-group>

      <b-form-group
        label="Address finder:"
        label-for="address-finder"
        class="d-flex align-items-center"
        label-class="mr-2 mb-1"
      >
        <b-form-checkbox id="address-finder" v-model="settings.addressFinder" switch></b-form-checkbox>
      </b-form-group>

      <div class="text-right">
        <b-button variant="outlined-primary" size="sm" to="/dashboard">Cancel</b-button>
        <b-button type="submit" variant="primary" size="sm" :disabled="loading">Submit</b-button>
      </div>
    </b-form>
  </validation-observer>
</template>

<script>
import { mapState, mapMutations, mapActions } from 'vuex'
import { BIcon, BIconFiles } from 'bootstrap-vue'

export default {
  middleware: 'auth',
  meta: {
    requiresAuth: true
  },
  components: {
    BIcon, BIconFiles
  },
  props: {
    isEdit: {
      type: Boolean,
      default: false
    }
  },
  data: () => ({
    loading: false,
    settings: {
      name: '',
      url: '',
      description: '',
      templateId: null,
      seo: [],
      distanceRings: [{
        on: false,
        miles: 0
      }, {
        on: false,
        miles: 0
      }, {
        on: false,
        miles: 0
      }],
      drivetimeRings: [{
        on: false,
        text: 'Walking'
      }, {
        on: false,
        text: 'Driving'
      }],
      addressFinder: false,
      basemapSelector: false
    }
  }),
  computed: {
    ...mapState('map', ['templates', 'error'])
  },
  async mounted () {
    this.CLEAR_ERROR()
    this.getMapTemplates()
    if (this.isEdit) {
      try {
        const response = await this.getMap(this.$route.params.id)

        this.settings.name = response.data.map.title
        this.settings.url = `${window.location.origin}/maps/${this.$route.params.id}/preview`
        this.settings.description = response.data.map.description
        this.settings.templateId = response.data.map.template_id
        this.settings.seo = JSON.parse(response.data.map.seo_keywords)
        this.settings.basemapSelector = response.data.map.map_selector
      } catch (err) {
        console.log(err)
      }
    }
  },
  methods: {
    ...mapActions('map', ['createMap', 'updateMap', 'getMapTemplates', 'getMap']),
    ...mapMutations('map', ['CLEAR_ERROR']),
    getValidationState({ dirty, validated, valid = null }) {
      return dirty || validated ? valid : null
    },
    async onSubmit() {
      this.CLEAR_ERROR()

      try {
        this.loading = true
        let response

        if (this.isEdit) {
          response = await this.updateMap({ ...this.settings, id: this.$route.params.id })
          this.$emit('updated', response.data.map)
        } else {
          response = await this.createMap(this.settings)
          this.$emit('created', response.data.map)
        }

      } catch (err) {
        console.log(err)
      } finally {
        this.loading = false
      }
    }
  }
}
</script>
