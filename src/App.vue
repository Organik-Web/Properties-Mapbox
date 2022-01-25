<template>

  <div id="property-finder-app">
      <div class="active loading-pop-up" v-if="showLoading"></div>

      <PropertiesMapBox v-if="(Object.keys(propertiesJson).length > 0)"
      :propertiesJson="propertiesJson"
      :rentalsJson="rentalsJson"
      :icons="icons"
      :propertySingle="propertySingle"
      :locationCoordinates="locationCoordinates"
      :getSingleProperty="getSingleProperty"
      :property="property"
      @isLoaded="isLoaded"
      />
  </div>
</template>

<script>
import axios from 'axios';
import PropertiesMapBox from '@/components/PropertiesMapBox';

export default {
  name: 'App',
  components: {
    PropertiesMapBox,
  },
  data() {
    return {
     propertiesJson: {},
     rentalsJson: {},
     icons: {},
     showLoading: true,
     property: {},
     locationCoordinates: null,
     isMobile: null
    }
  },
  mounted() {
    let element = document.querySelector('#post-id');
    let coordinates = document.querySelector('#post-coordinates');
    // If is property single
    if ( element ) {
      this.propertySingle = true;
      axios.get(`/wp-json/wp/v2/${element.dataset.type}/${element.dataset.post}`)
        .then((r) => this.propertiesJson = this.convertToGeoJsonSingle(r.data));
    }
    // If is locations single
    else if ( coordinates ) {
      this.propertySingle = false;
      let arr = coordinates.dataset.coordinates.split(/,| /);
      this.locationCoordinates = [parseFloat(arr[2]), parseFloat(arr[0])];
      axios.get('/wp-json/wp/v2/validproperties/coordinates')
        .then((r) => this.propertiesJson = this.convertToGeoJson(r.data));
      axios.get('/wp-json/wp/v2/validrentals/coordinates')
        .then((r) => this.rentalsJson = this.convertToGeoJson(r.data));
      }
      // If is full mapbox
      else {
      this.propertySingle = false;
      axios.get('/wp-json/wp/v2/validproperties/coordinates')
        .then((r) => this.propertiesJson = this.convertToGeoJson(r.data));
      axios.get('/wp-json/wp/v2/validrentals/coordinates')
        .then((r) => this.rentalsJson = this.convertToGeoJson(r.data));
    }

    axios.get('/wp-content/themes/HKY/images/property-icons/shower.svg')
      .then((r) => this.icons.showerIcon = r.data);
    axios.get('/wp-content/themes/HKY/images/property-icons/bed.svg')
      .then((r) => this.icons.bedIcon = r.data);
    axios.get('/wp-content/themes/HKY/images/property-icons/car.svg')
      .then((r) => this.icons.carIcon = r.data);
    axios.get('/wp-content/themes/HKY/images/property-icons/land.svg')
      .then((r) => this.icons.landIcon = r.data);
  },
  methods: {
    convertToGeoJsonSingle(data) {
      let arr = data.metadata.property_address_coordinates[0].split(/,| /);
      const geojson = {
        type: "FeatureCollection",
        features: [{
            id: data.id,
            type: "Feature",
            geometry: {
              type: "Point",
              coordinates: [parseFloat(arr[1]), parseFloat(arr[0]) ]
            }
          }]
      };
      return geojson;
    },
    convertToGeoJson(data) {
      const geojson = {
        type: "FeatureCollection",
        features: data.map(item => {
          let arr = item.postMeta.property_address_coordinates[0].split(/,| /);
          return {
            id: item.ID,
            type: "Feature",
            geometry: {
              type: "Point",
              coordinates: [parseFloat(arr[1]), parseFloat(arr[0]) ]
            }
          };
        })
      };
      return geojson;
    },
    getSingleProperty(feature, type) {
    return axios({
      url:  `/wp-json/wp/v2/${type}/${feature.id}?_embed`,
      method: 'get',
      timeout: 8000,
      headers: {
          'Content-Type': 'application/json',
      }
    })
    .then(res => res.data)
    .catch (err => console.error(err))
    },
    isLoaded() {
      this.showLoading = false
    },
  },
};
</script>

<style scoped>

#property-finder-app {
  width: 100%;
}
</style>
