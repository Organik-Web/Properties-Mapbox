<template>
  <div class="map-container">
    <div id="map"></div>
  </div>
</template>

<script>
import mapboxgl from 'mapbox-gl';
import MapboxDraw from "@mapbox/mapbox-gl-draw";
import turf from 'turf';

export default {
  name: 'PropertiesMapBox',
  mounted() {
    this.initMap(this.propertySingle);
    this.$emit('isLoaded');
  },
  data() {
    return {
      map: null,
      draw: null,
      singleProperty: null,
      property: {},
      output: null,
      popup: new mapboxgl.Popup()
    }
  },
  computed: {
  },
  props: {
    isLoading: {
      type: Boolean
    },
    getSingleProperty: {
      type: Function
    },
    propertySingle: {
      type: Boolean,
    },
    locationCoordinates: {
      type: String,
    },
    propertiesJson: {
      type: Object,
    },
    rentalsJson: {
      type: Object,
    },
    icons: {
      type: Object,
    },
  },
    watch: {
  },
  methods: {
    initMap() {
      mapboxgl.accessToken = process.env.VUE_APP_MAP_ACCESS_TOKEN;
      let zoom = 9
      let style = 'mapbox://styles/mapbox/light-v10?optimize=true'
      let coordinates = null;

      // Set the zoom and style of single properties
      if (this.propertySingle) {
        zoom = 15
        style = 'mapbox://styles/mapbox/streets-v11?optimize=true'
      }

      // Set style of single locations
      if (this.locationCoordinates) {
        zoom = 13
        style = 'mapbox://styles/mapbox/streets-v11?optimize=true'
        coordinates = [this.locationCoordinates[0], this.locationCoordinates[1]]
      } else {
        coordinates = [this.propertiesJson.features[0].geometry.coordinates[0], this.propertiesJson.features[0].geometry.coordinates[1]]
      }

      // Initiate map
      this.map = new mapboxgl.Map({
        container: 'map',
        style: style,
        center: coordinates,
        zoom: zoom,
      });

      this.map.on('load', () => {
        this.map.addControl(new mapboxgl.NavigationControl(), 'top-right');
        this.map.addControl(
          new mapboxgl.ScaleControl({
            maxWidth: 100,
            unit: 'metric',
          }),
        );
        if ( this.propertySingle ) {
          this.map.loadImage(
            '/wp-content/themes/HKY/images/property-icons/location-pin.png',
            (error, image) => {
              if (error) throw error;
              this.map.addImage('cat', image);

              this.map.addSource('properties', {
                type: 'geojson',
                // Point to GeoJSON data. This example visualizes all M1.0+ properties
                data: this.propertiesJson,
                cluster: true,
                clusterMaxZoom: 14, // Max zoom to cluster points on
                clusterRadius: 100, // Radius of each cluster when clustering points (defaults to 50)
              });

              this.map.addLayer({
                id: 'property-unclustered-point',
                type: 'symbol',
                source: 'properties',
                filter: ['!', ['has', 'point_count']],
                layout: {
                  'icon-image': 'cat', // reference the image
                  'icon-size': 0.25
                }
              });
            })
          } else {
            this.map.addSource('properties', {
              type: 'geojson',
              // Point to GeoJSON data. This example visualizes all M1.0+ properties
              data: this.propertiesJson,
              cluster: true,
              clusterMaxZoom: 14, // Max zoom to cluster points on
              clusterRadius: 100, // Radius of each cluster when clustering points (defaults to 50)
            });

            this.map.addSource('rentals', {
              type: 'geojson',
              data: this.rentalsJson,
              cluster: true,
              clusterMaxZoom: 14, // Max zoom to cluster points on
              clusterRadius: 100, // Radius of each cluster when clustering points (defaults to 50)
            });

            this.map.addLayer({
              id: 'property-clusters',
              type: 'circle',
              source: 'properties',
              filter: ['has', 'point_count'],
              paint: {
                'circle-color': [
                  'step',
                  ['get', 'point_count'],
                  '#00539C',
                  100,
                  '#00539C',
                  750,
                  '#00539C',
                ],
                'circle-radius': [
                  'step',
                  ['get', 'point_count'],
                  20,
                  100,
                  30,
                  750,
                  40,
                ],
              },
            });

            this.map.addLayer({
              id: 'rental-clusters',
              type: 'circle',
              source: 'rentals',
              filter: ['has', 'point_count'],
              paint: {
                'circle-color': [
                  'step',
                  ['get', 'point_count'],
                  '#9DC4E7',
                  100,
                  '#9DC4E7',
                  750,
                  '#9DC4E7',
                ],
                'circle-radius': [
                  'step',
                  ['get', 'point_count'],
                  20,
                  100,
                  30,
                  750,
                  40,
                ],
              },
            });

            this.map.addLayer({
              id: 'property-cluster-count',
              type: 'symbol',
              source: 'properties',
              filter: ['has', 'point_count'],
              layout: {
                'text-field': '{point_count_abbreviated}',
                'text-font': ['DIN Offc Pro Medium', 'Arial Unicode MS Bold'],
                'text-size': 12,
              },
              paint: {
                'text-color': '#ffffff',
              }
            });

            this.map.addLayer({
              id: 'rental-cluster-count',
              type: 'symbol',
              source: 'rentals',
              filter: ['has', 'point_count'],
              layout: {
                'text-field': '{point_count_abbreviated}',
                'text-font': ['DIN Offc Pro Medium', 'Arial Unicode MS Bold'],
                'text-size': 12,
              },
              paint: {
                'text-color': '#ffffff',
              }
            });

            this.map.addLayer({
              id: 'property-unclustered-point',
              type: 'circle',
              source: 'properties',
              filter: ['!', ['has', 'point_count']],
              paint: {
                'circle-color': '#00539C',
                'circle-radius': 10,
                'circle-stroke-width': 1,
                'circle-stroke-color': '#fff',
              },
            });

            this.map.addLayer({
              id: 'rental-unclustered-point',
              type: 'circle',
              source: 'rentals',
              filter: ['!', ['has', 'point_count']],
              paint: {
                'circle-color': '#9DC4E7',
                'circle-radius': 10,
                'circle-stroke-width': 1,
                'circle-stroke-color': '#fff',
              },
            });

            // Click events
            this.map.on('click', 'rental-unclustered-point', (e) => {
              this.featurePopup(e, 'rental')
              this.map.flyTo({
                center: e.features[0].geometry.coordinates
              });
            });

            // Click events
            this.map.on('click', 'property-unclustered-point', (e) => {
              this.featurePopup(e, 'property')
              this.map.flyTo({
                center: e.features[0].geometry.coordinates
              });
            });

            // Click events
            this.map.on('touchend', 'property-unclustered-point', (e) => {
              this.featurePopup(e, 'property')
              this.map.flyTo({
                center: e.features[0].geometry.coordinates
              });
            });

            // Click events
            this.map.on('touchend', 'rental-unclustered-point', (e) => {
              this.featurePopup(e, 'rental')
              this.map.flyTo({
                center: e.features[0].geometry.coordinates
              });
            });

            // inspect a cluster on click
            this.map.on('click', 'property-clusters', (e) => {
              this.doInspectCluster(e, 'property')
            });

            // inspect a cluster on click
            this.map.on('click', 'rental-clusters', (e) => {
              this.doInspectCluster(e, 'rental')
            });

            // inspect a cluster on touch
            this.map.on('touchend', 'rental-clusters', (e) => {
              this.doInspectCluster(e, 'rental')
            });

            // inspect a cluster on touch
            this.map.on('touchend', 'property-clusters', (e) => {
              this.doInspectCluster(e, 'property')
            });

            // Set cursor styles
            this.map.on('mouseenter', 'property-clusters', () => {
              this.map.getCanvas().style.cursor = 'pointer';
            });

            // Set cursor styles
            this.map.on('mouseleave', 'property-clusters', () => {
              this.map.getCanvas().style.cursor = '';
            });

            // Set cursor styles
            this.map.on('mouseenter', 'rental-clusters', () => {
              this.map.getCanvas().style.cursor = 'pointer';
            });

            // Set cursor styles
            this.map.on('mouseleave', 'rental-clusters', () => {
              this.map.getCanvas().style.cursor = '';
            });

            // Setting mapbox draw properties
            this.draw = new MapboxDraw({
              displayControlsDefault: false,
              // Select which mapbox-gl-draw control buttons to add to the map.
              controls: {
              polygon: true,
              trash: true
              },
            });

            this.map.addControl(this.draw);

            // Setting mapbox draw modes
            this.map.on('draw.modechange', (e) => {
              const data = this.draw.getAll();
              if (this.draw.getMode() == 'draw_polygon') {
                var pids = []
                // ID of the added template empty feature
                const lid = data.features[data.features.length - 1].id
                data.features.forEach((f) => {
                  if (f.geometry.type === 'Polygon' && f.id !== lid) {
                    pids.push(f.id)
                  }
                })
                this.draw.delete(pids)
                this.map.getSource('properties').setData(this.propertiesJson);
                this.map.getSource('rentals').setData(this.rentalsJson);
              }
            });
            this.map.on('draw.create', (e) => {
              this.updateArea(e);
            });
            this.map.on('draw.delete', (e) => {
              this.updateArea(e);
            });
            this.map.on('draw.update', (e) => {
              this.updateArea(e);
            });
        }
          this.map.once('load', () => {
            this.map.resize()
          })
      });
    },
    doInspectCluster(e, type) {
      let layer = null;
      let source = null;

      if (type === 'rental') {
        layer = 'rental-clusters';
        source = 'rentals';
      }

      if (type === 'property') {
        layer = 'property-clusters';
        source = 'properties';
      }

      const features = this.map.queryRenderedFeatures(e.point, {
          layers: [layer],
      });

      const clusterId = features[0].properties.cluster_id;

      this.map.getSource(source).getClusterExpansionZoom(
        clusterId,
        (err, zoom) => {
          if (err) return;
          this.map.easeTo({
            center: features[0].geometry.coordinates,
            zoom,
          });
        },
      );
    },
    doLoadingPopUp(feature) {
      this.output = `<div class="card loading-pop-up"> </div>`;
      this.popup
        .setLngLat(feature.lngLat)
        .setHTML(this.output)
        .addTo(this.map);
      return this.output;
    },
    updateArea(e) {
      if (e.type === 'draw.delete') {
          this.map.getSource('properties').setData(this.propertiesJson);
          this.map.getSource('rentals').setData(this.rentalsJson);
      } else {
        const data = this.draw.getAll();
        let polygon = turf.polygon([data.features[0].geometry.coordinates[0]], { name: 'poly1'});

        // Here we are setting up new geojson feature colections to store the properties within a polygon
        let newPropertyJson = {
          type: "FeatureCollection",
          features: []
        }

        // Here we are setting up new geojson feature colections to store the rentals within a polygon
        let newRentalJson = {
          type: "FeatureCollection",
          features: []
        }

        // Here we are looping over the json element and adding features within the boundaries to the newly created feature collection
        this.propertiesJson.features.forEach(element => {
          let point = turf.point([element.geometry.coordinates[0], element.geometry.coordinates[1]]);
          if (turf.inside(point, polygon) === true ) {
            newPropertyJson.features.push(element)
          }
        });

        // Here we are looping over the json element and adding features within the boundaries to the newly created feature collection
        this.rentalsJson.features.forEach(element => {
          let point = turf.point([element.geometry.coordinates[0], element.geometry.coordinates[1]]);
          if (turf.inside(point, polygon) === true ) {
            newRentalJson.features.push(element)
          }

          // Here we are resetting the sources to the new JSON properties
          this.map.getSource('properties').setData(newPropertyJson);
          this.map.getSource('rentals').setData(newRentalJson);

        });
      }
    },
    async featurePopup(feature, type) {

      this.doLoadingPopUp(feature);
      let output
      const res = async () => {
        await this.getSingleProperty(feature.features[0], type)
          .then(res => {
            this.property = res;
            output = `
              <div class="card">
                <div class="card-body entry">
                  <article id="post">
                    <div class="entry-wrapper">
                      <a class="entry-thumb property-link"  href="${this.property.link}"  target="_self" style="background-image: url('${this.property._embedded['wp:featuredmedia']['0'].source_url}')"';>
                        <span class="ratio-sizer"></span>
                      </a>
                      <div class="entry-preview">
                        <div class="entry-preview-content">
                          <div class="entry-preview-header">
                            <p class="title">${this.property.title.rendered}</p>
                          </div>
                          <div class="property-meta-group">
                            <div class="property-meta property-price">
                              ${this.property.metadata.property_price_view[0]}
                            </div>`;
              output = ( this.property.metadata.property_bedrooms[0] ? output + `<div class="property-meta property-bedroom">
                              <div class="meta-thumb">
                                <i class="icon svg-icon orgnk-svg-icon">
                                  ${this.icons.bedIcon}
                                </i>
                              </div>
                              <span>${this.property.metadata.property_bedrooms}</span>
                            </div>` : output)
              output = (  this.property.metadata.property_bathrooms[0] ? output + `<div class="property-meta property-bathrooms">
                              <div class="meta-thumb">
                                <i class="icon svg-icon orgnk-svg-icon">
                                  ${this.icons.showerIcon}
                                </i>
                              </div>
                              <span>${this.property.metadata.property_bathrooms}</span>
                            </div>` : output);
                output = ( this.property.metadata.property_carport[0] || this.property.metadata.property_garage[0] ? output + `
                            <div class="property-meta property-garage">
                              <div class="meta-thumb">
                                <i class="icon svg-icon orgnk-svg-icon">
                                  ${this.icons.carIcon}
                                </i>
                              </div>
                              <span>
                              ${parseInt(this.property.metadata.property_garage[0]) + parseInt(this.property.metadata.property_carport[0])}
                                </span>
                            </div> ` : output);
              output = output +
                        `</div>
                      </div>
                    </div>
                  </article>
                </div>
              </div>
            `;
            this.output = output;
            this.popup
            .setLngLat(feature.lngLat)
            .setHTML(this.output)
            .addTo(this.map);
          return this.output;
          });
        }
      return res();
    },
  },
};
</script>

<style>
</style>
