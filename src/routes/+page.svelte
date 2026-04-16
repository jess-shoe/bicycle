<script>
	import { onMount } from "svelte";
	import * as d3 from "d3";
	import mapboxgl from "mapbox-gl";
	import "mapbox-gl/dist/mapbox-gl.css";

	mapboxgl.accessToken = "pk.eyJ1IjoianRzaG9lIiwiYSI6ImNtbzEzdTFuMjBldnYycG90c2g0ZTZ0YWoifQ.9l9m3G0VYGapExZ2vz14dg";

	const BOSTON_LANES_URL =
		"https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D";

	const CAMBRIDGE_LANES_URL =
		"https://www.cambridgema.gov/GIS/gisdata/vector gis data/bike_facilities";

	const STATIONS_URL =
		"https://vis-society.github.io/labs/9/data/bluebikes-stations.csv";

	const TRIPS_URL =
		"https://vis-society.github.io/labs/9/data/bluebikes-traffic-2024-03.csv";

	let map;
	let stations = [];
	let trips = [];
	let filteredStations = [];

	let arrivals = new Map();
	let departures = new Map();
	let filteredArrivals = new Map();
	let filteredDepartures = new Map();

	let radiusScale = d3.scaleSqrt().domain([0, 1]).range([0, 25]);

	let mapViewChanged = 0;
	let timeFilter = -1;

	const departuresByMinute = Array.from({ length: 1440 }, () => []);
	const arrivalsByMinute = Array.from({ length: 1440 }, () => []);

	let selectedStation = null;
	let isochrone = null;

	const urlBase = "https://api.mapbox.com/isochrone/v1/mapbox/";
	const profile = "cycling";
	const minutes = [5, 10, 15, 20];
	const contourColors = ["03045e", "0077b6", "00b4d8", "90e0ef"];

	const stationFlow = d3.scaleQuantize().domain([0, 1]).range([0, 0.5, 1]);

	$: timeFilterLabel =
		timeFilter === -1
			? "(any time)"
			: new Date(0, 0, 0, 0, timeFilter).toLocaleString("en", {
					timeStyle: "short"
				});

	$: filteredDepartures =
		timeFilter === -1
			? departures
			: d3.rollup(
					filterByMinute(departuresByMinute, timeFilter),
					(v) => v.length,
					(d) => d.start_station_id
				);

	$: filteredArrivals =
		timeFilter === -1
			? arrivals
			: d3.rollup(
					filterByMinute(arrivalsByMinute, timeFilter),
					(v) => v.length,
					(d) => d.end_station_id
				);

	$: filteredStations = stations.map((station) => {
		const id = station.Number;
		const arr = filteredArrivals.get(id) ?? 0;
		const dep = filteredDepartures.get(id) ?? 0;

		return {
			...station,
			arrivals: arr,
			departures: dep,
			totalTraffic: arr + dep
		};
	});

	$: radiusScale = d3
		.scaleSqrt()
		.domain([0, d3.max(filteredStations, (d) => d.totalTraffic) || 0])
		.range(timeFilter === -1 ? [0, 25] : [3, 30]);

	$: if (selectedStation) {
		getIso(+selectedStation.Long, +selectedStation.Lat);
	} else {
		isochrone = null;
	}

	onMount(async () => {
		await loadStations();
		await loadTrips();
		await initMap();
	});

	async function loadStations() {
		stations = await d3.csv(STATIONS_URL, (d) => ({
			...d,
			Lat: +d.Lat,
			Long: +d.Long,
			"Total Docks": +d["Total Docks"]
		}));
	}

	async function loadTrips() {
		trips = await d3.csv(TRIPS_URL).then((rows) => {
			for (const trip of rows) {
				trip.started_at = new Date(trip.started_at);
				trip.ended_at = new Date(trip.ended_at);

				const startedMinutes = minutesSinceMidnight(trip.started_at);
				const endedMinutes = minutesSinceMidnight(trip.ended_at);

				departuresByMinute[startedMinutes].push(trip);
				arrivalsByMinute[endedMinutes].push(trip);
			}
			return rows;
		});

		departures = d3.rollup(
			trips,
			(v) => v.length,
			(d) => d.start_station_id
		);

		arrivals = d3.rollup(
			trips,
			(v) => v.length,
			(d) => d.end_station_id
		);

		stations = stations.map((station) => {
			const id = station.Number;
			const arr = arrivals.get(id) ?? 0;
			const dep = departures.get(id) ?? 0;

			return {
				...station,
				arrivals: arr,
				departures: dep,
				totalTraffic: arr + dep
			};
		});
	}

	async function initMap() {
		map = new mapboxgl.Map({
			container: "map",
			style: "mapbox://styles/mapbox/streets-v12",
			center: [-71.09415, 42.36027],
			zoom: 12,
			minZoom: 10,
			maxZoom: 18
		});

		map.on("move", () => {
			mapViewChanged += 1;
		});

		await new Promise((resolve) => map.on("load", resolve));

        addBikeLaneLayers();
        mapViewChanged += 1;
	}

	function addBikeLaneLayers() {
		map.addSource("boston_route", {
			type: "geojson",
			data: BOSTON_LANES_URL
		});

		map.addLayer({
			id: "boston-lanes",
			type: "line",
			source: "boston_route",
			paint: {
				"line-color": "#32d400",
				"line-width": 3,
				"line-opacity": 0.35
			}
		});

		map.addSource("cambridge_route", {
			type: "geojson",
			data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson"
		});

		map.addLayer({
			id: "cambridge-lanes",
			type: "line",
			source: "cambridge_route",
			paint: {
				"line-color": "#32d400",
				"line-width": 3,
				"line-opacity": 0.35
			}
		});
	}

	function getCoords(station) {
		if (!map) return { cx: 0, cy: 0 };
		const point = new mapboxgl.LngLat(+station.Long, +station.Lat);
		const { x, y } = map.project(point);
		return { cx: x, cy: y };
	}

	function minutesSinceMidnight(date) {
		return date.getHours() * 60 + date.getMinutes();
	}

	function filterByMinute(tripsByMinute, minute) {
		if (minute === -1) {
			return tripsByMinute.flat();
		}

		const minMinute = (minute - 60 + 1440) % 1440;
		const maxMinute = (minute + 60) % 1440;

		if (minMinute > maxMinute) {
			const beforeMidnight = tripsByMinute.slice(minMinute);
			const afterMidnight = tripsByMinute.slice(0, maxMinute);
			return beforeMidnight.concat(afterMidnight).flat();
		}

		return tripsByMinute.slice(minMinute, maxMinute).flat();
	}

	async function getIso(lon, lat) {
		const base = `${urlBase}${profile}/${lon},${lat}`;
		const params = new URLSearchParams({
			contours_minutes: minutes.join(","),
			contours_colors: contourColors.join(","),
			polygons: "true",
			access_token: mapboxgl.accessToken
		});

		const url = `${base}?${params.toString()}`;
		const query = await fetch(url, { method: "GET" });
		isochrone = await query.json();

		// Force SVG redraw after async fetch completes.
		mapViewChanged += 1;
	}

	function geoJSONPolygonToPath(feature) {
		if (!map) return "";

		const path = d3.path();

		// Handle both Polygon and MultiPolygon so the path always renders.
		if (feature.geometry.type === "Polygon") {
			const rings = feature.geometry.coordinates;

			for (const ring of rings) {
				for (let i = 0; i < ring.length; i++) {
					const [lng, lat] = ring[i];
					const { x, y } = map.project([lng, lat]);
					if (i === 0) path.moveTo(x, y);
					else path.lineTo(x, y);
				}
				path.closePath();
			}
		} else if (feature.geometry.type === "MultiPolygon") {
			for (const polygon of feature.geometry.coordinates) {
				for (const ring of polygon) {
					for (let i = 0; i < ring.length; i++) {
						const [lng, lat] = ring[i];
						const { x, y } = map.project([lng, lat]);
						if (i === 0) path.moveTo(x, y);
						else path.lineTo(x, y);
					}
					path.closePath();
				}
			}
		}

		return path.toString();
	}

	function departureRatio(station) {
		if (!station.totalTraffic) return 0.5;
		return stationFlow(station.departures / station.totalTraffic);
	}

	function toggleSelectedStation(station) {
		selectedStation =
			selectedStation?.Number !== station?.Number ? station : null;

		// Immediate rerender so highlight/dimming shows instantly.
		mapViewChanged += 1;
	}
</script>

<svelte:head>
	<title>BikeWatch</title>
</svelte:head>

<header>
	<h1>🚲 BikeWatch</h1>

	<label class="time-filter">
		Filter by time:
		<input type="range" min="-1" max="1440" bind:value={timeFilter} />
		{#if timeFilter !== -1}
			<time>{timeFilterLabel}</time>
		{:else}
			<em>(any time)</em>
		{/if}
	</label>
</header>

<p class="description">
	Explore BlueBike traffic across Boston and Cambridge by station, time of day,
	and travel reachability.
</p>

<div id="map">
	<svg aria-label="BlueBike stations and isochrones overlay">
		{#key mapViewChanged}
			{#if isochrone}
				{#each isochrone.features as feature}
					<path
						d={geoJSONPolygonToPath(feature)}
						fill={feature.properties.fillColor}
						fill-opacity="0.2"
						stroke="#000000"
						stroke-opacity="0.5"
						stroke-width="1"
					>
						<title>{feature.properties.contour} minutes of biking</title>
					</path>
				{/each}
			{/if}

			{#each filteredStations as station}
				<circle
					{...getCoords(station)}
					r={radiusScale(station.totalTraffic)}
					style={`--departure-ratio: ${departureRatio(station)}`}
					class:selected={station.Number === selectedStation?.Number}
					on:click={() => toggleSelectedStation(station)}
				>
					<title>
						{station.NAME}: {station.totalTraffic} trips ({station.departures}
						departures, {station.arrivals} arrivals)
					</title>
				</circle>
			{/each}
		{/key}
	</svg>
</div>

<div class="legend" aria-label="traffic flow legend">
	<div class="legend-block" style="--departure-ratio: 1">More departures</div>
	<div class="legend-block" style="--departure-ratio: 0.5">Balanced</div>
	<div class="legend-block" style="--departure-ratio: 0">More arrivals</div>
</div>

<style>
	@import url("$lib/global.css");

	header {
		display: flex;
		gap: 1rem;
		align-items: baseline;
		flex-wrap: wrap;
	}

	h1 {
		margin: 0;
	}

	.description {
		margin: 0.5rem 0 1rem 0;
	}

	.time-filter {
		margin-left: auto;
		display: flex;
		flex-direction: column;
		gap: 0.25rem;
		min-width: 240px;
	}

	.time-filter time,
	.time-filter em {
		display: block;
	}

	.time-filter em {
		color: #666;
		font-style: italic;
	}

	#map {
		position: relative;
		flex: 1;
		min-height: 70vh;
		border-radius: 10px;
		overflow: hidden;
	}

	#map svg {
		position: absolute;
		inset: 0;
		z-index: 1;
		width: 100%;
		height: 100%;
		pointer-events: none;
	}

	#map svg circle,
	.legend > div {
		--color-departures: steelblue;
		--color-arrivals: darkorange;
		--color: color-mix(
			in oklch,
			var(--color-departures) calc(100% * var(--departure-ratio)),
			var(--color-arrivals)
		);

		fill: var(--color);
	}

	#map svg circle {
		fill-opacity: 0.7;
		stroke: white;
		stroke-width: 1;
		pointer-events: auto;
		transition:
			opacity 0.2s ease,
			stroke 0.2s ease,
			stroke-width 0.2s ease,
			r 0.2s ease;
		cursor: pointer;
	}

	#map svg circle.selected {
		stroke: black;
		stroke-width: 2.5;
	}

	#map svg path {
		pointer-events: auto;
	}

	#map svg:has(circle.selected) circle:not(.selected) {
		opacity: 0.3;
	}

	.legend {
		display: flex;
		gap: 2px;
		margin-block: 1rem;
		border-radius: 8px;
		overflow: hidden;
		box-shadow: 0 1px 4px rgba(0, 0, 0, 0.12);
	}

	.legend > div {
		flex: 1;
		padding: 0.75rem 1rem;
		color: white;
		font-weight: 600;
		text-shadow: 0 1px 1px rgba(0, 0, 0, 0.25);
	}

	.legend > div:nth-child(1) {
		text-align: left;
	}

	.legend > div:nth-child(2) {
		text-align: center;
	}

	.legend > div:nth-child(3) {
		text-align: right;
	}
</style>