<template>
  <main class="w-screen h-screen">
    <v-map class="w-full h-full" :options="state.map" @loaded="onMapLoaded">
      <template v-if="loaded">
        <v-layer-mapbox-geojson
          v-if="loaded"
          :source="cluster.source"
          :source-id="'earthquakes-0'"
          :layer="cluster.circleLayer"
          :layer-id="cluster.circleLayer.id"
        />
        <v-layer-mapbox-geojson
          v-if="loaded"
          :source="cluster.source"
          :source-id="'earthquakes-1'"
          :layer="cluster.countLayer"
          :layer-id="cluster.countLayer.id"
        />
        <v-layer-mapbox-geojson
          v-if="loaded"
          :source="cluster.source"
          :source-id="'earthquakes-2'"
          :layer="cluster.pointLayer"
          :layer-id="cluster.pointLayer.id"
        />
      </template>
    </v-map>
  </main>
</template>

<script lang="ts">
  import type {
    LayerSpecification,
    GeoJSONSourceSpecification,
    Map,
    MapOptions,
  } from 'maplibre-gl';
  import VMap, { VLayerMapboxGeojson } from 'v-mapbox';
  import { computed, reactive, ref, defineComponent } from 'vue';

  export default defineComponent({
    name: 'App',
    components: {
      VMap,
      VLayerMapboxGeojson,
    },
    setup() {
      const state = reactive({
        map: {
          container: 'map',
          style:
            'https://basemaps.cartocdn.com/gl/dark-matter-gl-style/style.json',
          center: [79.0882, 21.1458],
          zoom: 4,
        } as MapOptions,
        ui: {
          loaded: false,
          styleChanged: false,
          tilesLoaded: false,
        },
      });

      const cluster = ref({
        source: {
          type: 'geojson',
          data: 'https://maplibre.org/maplibre-gl-js-docs/assets/earthquakes.geojson',
        } as GeoJSONSourceSpecification,
        circleLayer: {
          id: 'earthquakes-heat',
          type: 'heatmap',
          source: 'earthquakes',
          maxzoom: 9,
          paint: {
            // Increase the heatmap weight based on frequency and property magnitude
            'heatmap-weight': [
              'interpolate',
              ['linear'],
              ['get', 'mag'],
              0,
              0,
              6,
              1,
            ],
            // Increase the heatmap color weight weight by zoom level
            // heatmap-intensity is a multiplier on top of heatmap-weight
            'heatmap-intensity': [
              'interpolate',
              ['linear'],
              ['zoom'],
              0,
              1,
              9,
              3,
            ],
            // Color ramp for heatmap.  Domain is 0 (low) to 1 (high).
            // Begin color ramp at 0-stop with a 0-transparancy color
            // to create a blur-like effect.
            'heatmap-color': [
              'interpolate',
              ['linear'],
              ['heatmap-density'],
              0,
              'rgba(33,102,172,0)',
              0.2,
              'rgb(103,169,207)',
              0.4,
              'rgb(209,229,240)',
              0.6,
              'rgb(253,219,199)',
              0.8,
              'rgb(239,138,98)',
              1,
              'rgb(178,24,43)',
            ],
            // Adjust the heatmap radius by zoom level
            'heatmap-radius': [
              'interpolate',
              ['linear'],
              ['zoom'],
              0,
              2,
              9,
              20,
            ],
            // Transition from heatmap to circle layer by zoom level
            'heatmap-opacity': [
              'interpolate',
              ['linear'],
              ['zoom'],
              7,
              1,
              9,
              0,
            ],
          },
        } as LayerSpecification,
        countLayer: {
          id: 'cluster-count',
          type: 'symbol',
          source: 'earthquakes',
          filter: ['has', 'point_count'],
          layout: {
            'text-field': '{point_count_abbreviated}',
            'text-font': ['DIN Offc Pro Medium', 'Arial Unicode MS Bold'],
            'text-size': 12,
          },
        } as LayerSpecification,
        pointLayer: {
          id: 'earthquakes-point',
          type: 'circle',
          source: 'earthquakes',
          minzoom: 7,
          paint: {
            // Size circle radius by earthquake magnitude and zoom level
            'circle-radius': [
              'interpolate',
              ['linear'],
              ['zoom'],
              7,
              ['interpolate', ['linear'], ['get', 'mag'], 1, 1, 6, 4],
              16,
              ['interpolate', ['linear'], ['get', 'mag'], 1, 5, 6, 50],
            ],
            // Color circle by earthquake magnitude
            'circle-color': [
              'interpolate',
              ['linear'],
              ['get', 'mag'],
              1,
              'rgba(33,102,172,0)',
              2,
              'rgb(103,169,207)',
              3,
              'rgb(209,229,240)',
              4,
              'rgb(253,219,199)',
              5,
              'rgb(239,138,98)',
              6,
              'rgb(178,24,43)',
            ],
            'circle-stroke-color': 'white',
            'circle-stroke-width': 1,
            // Transition from heatmap to circle layer by zoom level
            'circle-opacity': ['interpolate', ['linear'], ['zoom'], 7, 0, 8, 1],
          },
        } as LayerSpecification,
      });

      const loaded = computed(() => state.ui.loaded || state.ui.styleChanged);

      const onMapLoaded = (map: Map) => {
        const events = [
          'idle',
          'moveend',
          'touchend',
          'style.load',
          'sourcedata',
          'sourcedataloading',
        ];
        events.forEach((event) => {
          map.on(event, () => {
            if (event === 'sourcedata' || event === 'sourcedataloading') {
              const waiting = () => {
                if (!map.areTilesLoaded()) {
                  state.ui.tilesLoaded = false;
                  setTimeout(waiting, 200);
                } else {
                  state.ui.tilesLoaded = true;
                }
              };
              waiting();
            }
            if (event === 'style.load') {
              const waiting = () => {
                if (!map.isStyleLoaded()) {
                  state.ui.styleChanged = false;
                  setTimeout(waiting, 200);
                } else {
                  state.ui.styleChanged = true;
                }
              };
              waiting();
            }
          });
        });
        state.ui.loaded = true;
      };

      return {
        state,
        cluster,
        loaded,
        onMapLoaded,
      };
    },
  });
</script>

<style lang="scss">
  @import 'maplibre-gl/dist/maplibre-gl.css';
  @import 'v-mapbox/dist/v-mapbox.css';
  html,
  body {
    margin: 0;
  }
  #app {
    font-family: Avenir, Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    text-align: center;
    color: #2c3e50;
  }
  .w-screen {
    width: 100vw;
  }
  .h-screen {
    height: 100vh;
  }
  .h-full {
    height: 100%;
  }
  .w-full {
    width: 100%;
  }
</style>
