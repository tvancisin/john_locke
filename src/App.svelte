<script>
  import { onMount } from "svelte";
  import * as d3 from "d3";
  import { locations } from "./utils.js";
  import mapboxgl from "mapbox-gl";
  import "mapbox-gl/dist/mapbox-gl.css";

  let map,
    start = 1400,
    end = 1900,
    width,
    height,
    mapContainer,
    current_university,
    hoveredId = null;

  mapboxgl.accessToken =
    "pk.eyJ1IjoidG9tYXN2YW5jaXNpbiIsImEiOiJjbTN1OXUzODUwZTEwMnFxdHd5NzA3cmNuIn0.vz2M0cTMfPvLAQ-wKMKbQA";

  const margin = { top: 20, bottom: 20 };
  $: innerHeight = height - margin.top - margin.bottom;
  $: yScale = d3.scaleLinear().domain([start, end]).range([0, innerHeight]);
  $: ticks = d3.range(start, end + 1, 50);

  // load map
  onMount(() => {
    map = new mapboxgl.Map({
      container: mapContainer,
      style: "mapbox://styles/mapbox/light-v11",
      projection: "mercator",
      center: [11, 50],
      zoom: 4.7,
    });

    map.on("load", () => {
      map.addSource("germany", {
        type: "raster",
        url: "mapbox://tomasvancisin.1mwrt593",
      });

      // adding the historical map
      map.addLayer({
        id: "germany",
        source: "germany",
        type: "raster",
      });

      // second historical layer
      map.addSource("kaliningrad", {
        type: "raster",
        url: "mapbox://tomasvancisin.2cefmsvl",
      });

      // adding the historical map
      map.addLayer({
        id: "kaliningrad",
        source: "kaliningrad",
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
            yearFounded: location.yearFounded,
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
          "text-offset": [0, 1.7],
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
        const [lng, lat] = feature.geometry.coordinates;

        current_university = feature;

        // Open info panel
        d3.select("#location-info").style("right", "0px");
        d3.select("#header").html(`
      <h2>${feature.properties.name}</h2>
    `);

        // Calculate offset for 40% panel
        const panelWidth = window.innerWidth * 0.4;
        const offsetX = -panelWidth / 2; // move map center left

        map.flyTo({
          center: [lng, lat],
          zoom: 9,
          offset: [offsetX, 0],
          speed: 0.8,
          curve: 1.4,
          essential: true,
        });
      });
    });
  });

  // close details panel
  function closeDetails() {
    d3.select("#location-info").style("right", "-40%");
    map.flyTo({
      center: [11, 50],
      zoom: 5,
      speed: 0.8,
      curve: 1.4,
      essential: true,
    });
  }

  $: console.log(current_university);
</script>

<main>
  <div id="map" bind:this={mapContainer}></div>
  <div id="location-info">
    <div id="header"></div>
    <button id="close-details" on:click={closeDetails}>
      <i class="fa fa-close" aria-hidden="true"></i>
    </button>
    <div id="details" bind:clientWidth={width} bind:clientHeight={height}>
      <svg {width} {height}>
        <g transform={`translate(${50}, ${margin.top})`}>
          <!-- main line -->
          <line
            x1="0"
            x2="0"
            y1="0"
            y2={innerHeight}
            stroke="black"
            stroke-width="2"
          />

          <!-- ticks + labels -->
          {#each ticks as year}
            <g transform={`translate(0, ${yScale(year)})`}>
              <line x1="-6" x2="6" y1="0" y2="0" stroke="black" />
              <text
                x="-10"
                y="4"
                text-anchor="end"
                font-family="Montserrat"
                font-size="11"
              >
                {year}
              </text>
            </g>
          {/each}

          {#if current_university}
            <circle
              cx="0"
              cy={yScale(current_university.properties.yearFounded)}
              r="7"
              fill="black"
              stroke="black"
            />
            <text
              x="15"
              y={yScale(current_university.properties.yearFounded) + 4}
              font-family="Montserrat"
              font-size="14"
            >
              University Established
            </text>
          {/if}
        </g>
      </svg>
    </div>
  </div>
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

  #location-info {
    position: absolute;
    top: 0;
    right: -40%;
    width: 40%;
    height: 100vh;

    display: flex;
    flex-direction: column;

    background: white;
    padding: 10px;
    border-radius: 5px;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);

    font-family: "Montserrat", sans-serif;
    font-size: 14px;
    z-index: 10;
    transition: right 0.3s ease;
    box-sizing: border-box;
  }

  #header {
    position: relative;
    text-align: center;
  }

  #close-details {
    position: absolute;
    top: 2px;
    right: 4px;
    background: none;
    color: #000;
    border: none;
    padding: 2px 10px;
    border-radius: 2px;
    font-size: 2em;
    cursor: pointer;
    font-family: Montserrat;
    transition: border 0.3s ease;
  }

  #details {
    flex: 1;
    overflow-y: auto;
  }
  svg {
    display: block;
    overflow: hidden;
  }
</style>
