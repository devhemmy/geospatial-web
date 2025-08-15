<script lang="ts">
	import { PUBLIC_MAPTILER_API_KEY } from '$env/static/public';
	import { onMount } from 'svelte';
	import maplibregl, { Map } from 'maplibre-gl';
	import 'maplibre-gl/dist/maplibre-gl.css';

	let mapContainer: HTMLElement;
	let map: Map;

	onMount(async () => {
		// Make the onMount function async
		// 1. Dynamically import and register the COG protocol
		// This happens only in the browser, avoiding SSR errors.
		const { cogProtocol } = await import('@geomatico/maplibre-cog-protocol');
		maplibregl.addProtocol('cog', cogProtocol);

		const apiKey = PUBLIC_MAPTILER_API_KEY;

		map = new Map({
			container: mapContainer,
			style: `https://api.maptiler.com/maps/streets-v2/style.json?key=${apiKey}`,
			// Updated center/zoom to focus on the COG's location
			center: [11.39831, 47.26244],
			zoom: 14
		});

		map.on('load', () => {
			// The Austria boundary layers are still here
			map.addSource('austria-boundary', {
				type: 'geojson',
				data: '/austria.geojson'
			});
			map.addLayer({
				id: 'austria-outline',
				type: 'line',
				source: 'austria-boundary',
				paint: {
					'line-color': '#804000',
					'line-width': 2
				}
			});

			// 2. Add the COG data source using the new protocol
			map.addSource('cog-source', {
				type: 'raster',
				url: 'cog://https://maplibre.org/maplibre-gl-js/docs/assets/cog.tif',
				tileSize: 256
			});

			// 3. Add the raster layer to display the COG
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
