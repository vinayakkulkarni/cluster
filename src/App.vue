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
    AnyLayer,
    GeoJSONSourceRaw,
    Map,
    MapboxOptions,
  } from 'mapbox-gl';
  import VMap, { VLayerMapboxGeojson } from 'v-mapbox';
  import { computed, reactive, ref } from 'vue';

  export default {
    name: 'App',
    components: {
      VMap,
      VLayerMapboxGeojson,
    },
    setup() {
      const state = reactive({
        map: {
          container: 'map2',
          accessToken:
            'pk.eyJ1Ijoic29jaWFsZXhwbG9yZXIiLCJhIjoiREFQbXBISSJ9.dwFTwfSaWsHvktHrRtpydQ',
          style: 'mapbox://styles/mapbox/streets-v11?optimize=true',
          // style: "https://basemaps.cartocdn.com/gl/dark-matter-gl-style/style.json",
          center: [79.0882, 21.1458],
          zoom: 4,
          maxZoom: 22,
          crossSourceCollisions: false,
          failIfMajorPerformanceCaveat: false,
          attributionControl: false,
          preserveDrawingBuffer: true,
          hash: false,
          minPitch: 0,
          maxPitch: 60,
        } as MapboxOptions,
        ui: {
          loaded: false,
          styleChanged: false,
          tilesLoaded: false,
        },
      });
      const cluster = ref({
        source: {
          type: 'geojson',
          // Point to GeoJSON data. This example visualizes all M1.0+ earthquakes
          // from 12/22/15 to 1/21/16 as logged by USGS' Earthquake hazards program.
          data: 'https://docs.mapbox.com/mapbox-gl-js/assets/earthquakes.geojson',
          cluster: true,
          clusterMaxZoom: 14, // Max zoom to cluster points on
          clusterRadius: 50, // Radius of each cluster when clustering points (defaults to 50)
        } as GeoJSONSourceRaw,
        circleLayer: {
          id: 'clusters',
          type: 'circle',
          source: 'earthquakes',
          filter: ['has', 'point_count'],
          paint: {
            // Use step expressions (https://docs.mapbox.com/mapbox-gl-js/style-spec/#expressions-step)
            // with three steps to implement three types of circles:
            //   * Blue, 20px circles when point count is less than 100
            //   * Yellow, 30px circles when point count is between 100 and 750
            //   * Pink, 40px circles when point count is greater than or equal to 750
            'circle-color': [
              'step',
              ['get', 'point_count'],
              '#51bbd6',
              100,
              '#f1f075',
              750,
              '#f28cb1',
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
        } as AnyLayer,
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
        } as AnyLayer,
        pointLayer: {
          id: 'unclustered-point',
          type: 'circle',
          source: 'earthquakes',
          filter: ['!', ['has', 'point_count']],
          paint: {
            'circle-color': '#11b4da',
            'circle-radius': 4,
            'circle-stroke-width': 1,
            'circle-stroke-color': '#fff',
          },
        } as AnyLayer,
      });

      const loaded = computed(() => state.ui.loaded || state.ui.styleChanged);

      function onMapLoaded(map: Map) {
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
      }

      return {
        state,
        cluster,
        loaded,
        onMapLoaded,
      };
    },
  };
</script>

<style lang="scss">
  @import 'mapbox-gl/dist/mapbox-gl.css';
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
