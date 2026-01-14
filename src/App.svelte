<script>
  import { onMount } from "svelte";
  import { locations } from "./utils.js";
  import mapboxgl from "mapbox-gl";
  import "mapbox-gl/dist/mapbox-gl.css";

  let map,
    mapContainer,
    hoveredId = null;
  mapboxgl.accessToken =
    "pk.eyJ1IjoidG9tYXN2YW5jaXNpbiIsImEiOiJjbTN1OXUzODUwZTEwMnFxdHd5NzA3cmNuIn0.vz2M0cTMfPvLAQ-wKMKbQA";

  // load map
  onMount(() => {
    map = new mapboxgl.Map({
      container: mapContainer,
      style: "mapbox://styles/mapbox/light-v11",
      projection: "mercator",
      center: [11, 50],
      zoom: 5,
    });

    map.on("load", () => {
      map.addSource("portland", {
        type: "raster",
        url: "mapbox://tomasvancisin.1mwrt593",
      });

      // adding the historical map
      map.addLayer({
        id: "portland",
        source: "portland",
        type: "raster",
      });

      // create locations geojson
      const geojson = {
        type: "FeatureCollection",
        features: locations.map((location, i) => ({
          type: "Feature",
          id: i,
          geometry: {
            type: "Point",
            coordinates: [location.coordinates.lng, location.coordinates.lat],
          },
          properties: {
            name: location.name,
          },
        })),
      };

      map.addSource("locations", {
        type: "geojson",
        data: geojson,
      });

      map.addLayer({
        id: "location-circles",
        type: "circle",
        source: "locations",
        paint: {
          "circle-radius": [
            "interpolate",
            ["linear"],
            ["zoom"],

            5,
            [
              "case",
              ["boolean", ["feature-state", "hover"], false],
              7, // hover
              5, // normal
            ],

            10,
            ["case", ["boolean", ["feature-state", "hover"], false], 12, 9],

            15,
            ["case", ["boolean", ["feature-state", "hover"], false], 16, 12],
          ],

          "circle-color": "black",
          "circle-opacity": 0.6,
          "circle-stroke-color": "#ffffff",
          "circle-stroke-width": 1.5,
          "circle-radius-transition": {
            delay: 0,
          },
        },
      });

      // interaction
      map.on("mousemove", "location-circles", (e) => {
        if (e.features.length === 0) return;

        const featureId = e.features[0].id;
        if (hoveredId !== null) {
          map.setFeatureState(
            { source: "locations", id: hoveredId },
            { hover: false },
          );
        }

        hoveredId = featureId;
        map.setFeatureState(
          { source: "locations", id: hoveredId },
          { hover: true },
        );
      });

      map.on("mouseleave", "location-circles", () => {
        if (hoveredId !== null) {
          map.setFeatureState(
            { source: "locations", id: hoveredId },
            { hover: false },
          );
        }
        hoveredId = null;
      });

      // change cursor
      map.on("mouseenter", "location-circles", () => {
        map.getCanvas().style.cursor = "pointer";
      });

      map.on("mouseleave", "location-circles", () => {
        map.getCanvas().style.cursor = "";
      });

      // uni labels
      map.addLayer({
        id: "location-labels",
        type: "symbol",
        source: "locations",
        layout: {
          "text-field": ["get", "name"],
          "text-size": [
            "interpolate",
            ["linear"],
            ["zoom"],
            5,
            12,
            10,
            14,
            15,
            17,
          ],
          "text-font": ["Montserrat Bold"],
          "text-anchor": "bottom",
          "text-offset": [0, -0.4],
          "text-allow-overlap": true,
        },
        paint: {
          "text-halo-color": "#ffffff",
          "text-color": "#000000",
          "text-halo-width": 2,
          "text-halo-blur": 0.7,
        },
      });

      // popups
      map.on("click", "location-circles", (e) => {
        const feature = e.features[0];

        const popupHTML = `
        <div class="popup-content">
          <strong class="popup-title">${feature.properties.name}</strong>
          <p class="popup-text">
            Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. 
            Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. 
            Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. 
            Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.
          </p>
        </div>
      `;

        new mapboxgl.Popup({ offset: 15, maxWidth: "300px" }) // optional max width
          .setLngLat(feature.geometry.coordinates)
          .setHTML(popupHTML)
          .addTo(map);
      });

      // locations.forEach((location) => {
      //   new mapboxgl.Marker()
      //     .setLngLat([location.coordinates.lng, location.coordinates.lat])
      //     .setPopup(
      //       new mapboxgl.Popup({ offset: 25 }) // add popups
      //         .setHTML(`
      //             <h3>${location.name}</h3>
      //           `),
      //     )
      //     .addTo(map);
      // });
    });
  });
</script>

<main>
  <div id="map" bind:this={mapContainer}></div>
  <div id="location-info"></div>
</main>

<style>
  main,
  #map {
    width: 100%;
    height: 100vh;
  }
  :global(.mapboxgl-popup-content) {
    font-family: "Montserrat", sans-serif !important;
    font-weight: 350;
    font-size: 12px;
  }
  :global(.mapboxgl-popup-close-button) {
    font-family: "Montserrat", sans-serif !important;
    font-weight: 500;
    font-size: 20px;
  }
</style>
