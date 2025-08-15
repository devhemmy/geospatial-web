<script lang="ts">
	import { PUBLIC_MAPTILER_API_KEY } from '$env/static/public';
	import { onMount } from 'svelte';
	import maplibregl, { Map } from 'maplibre-gl';
	import 'maplibre-gl/dist/maplibre-gl.css';

	let mapContainer: HTMLElement;
	let map: Map;

	onMount(async () => {
		const apiKey = PUBLIC_MAPTILER_API_KEY;

		map = new Map({
			container: mapContainer,
			style: `https://api.maptiler.com/maps/streets-v2/style.json?key=${apiKey}`,
			center: [14.55, 47.52],
			zoom: 6.5
		});

		map.on('load', async () => {
			map.addSource('austria-boundary', {
				type: 'geojson',
				data: '/austria.geojson'
			});
			map.addLayer({
				id: 'austria-outline',
				type: 'line',
				source: 'austria-boundary',
				paint: { 'line-color': '#804000', 'line-width': 2 }
			});

			const cogUrl = 'https://maplibre.org/maplibre-gl-js/docs/assets/cog.tif';

			const tilejsonUrl = `https://titiler.xyz/cog/WebMercatorQuad/tilejson.json?url=${encodeURIComponent(cogUrl)}&bidx=1&bidx=2&bidx=3`;

			const response = await fetch(tilejsonUrl);
			const tileJson = await response.json();

			const bbox = tileJson.bounds;
			const bounds: [[number, number], [number, number]] = [
				[bbox[0], bbox[1]],
				[bbox[2], bbox[3]]
			];

			map.fitBounds(bounds, {
				padding: 50,
				duration: 3000
			});

			map.addSource('cog-source', {
				type: 'raster',
				url: tilejsonUrl,
				tileSize: 256
			});

			map.addLayer({
				id: 'cog-layer',
				type: 'raster',
				source: 'cog-source'
			});
		});

		return () => {
			map?.remove();
		};
	});
</script>

<main class="h-screen w-screen" bind:this={mapContainer} />
