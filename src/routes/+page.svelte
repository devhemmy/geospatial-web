<script lang="ts">
	import { PUBLIC_MAPTILER_API_KEY } from '$env/static/public';
	import { onMount } from 'svelte';
	import maplibregl, { Map } from 'maplibre-gl';
	import 'maplibre-gl/dist/maplibre-gl.css';

	let mapContainer: HTMLElement;
	let map: Map;

	onMount(() => {
		const apiKey = PUBLIC_MAPTILER_API_KEY;

		map = new Map({
			container: mapContainer,
			style: `https://api.maptiler.com/maps/streets-v2/style.json?key=${apiKey}`,
			center: [133.7751, -25.2744],
			zoom: 4
		});

		map.on('load', () => {
			map.addSource('australia-boundary', {
				type: 'geojson',
				data: '/australia.geojson'
			});

			map.addLayer({
				id: 'australia-fill',
				type: 'fill',
				source: 'australia-boundary',
				paint: {
					'fill-color': '#ff4500',
					'fill-opacity': 0.3
				}
			});

			map.addLayer({
				id: 'australia-outline',
				type: 'line',
				source: 'australia-boundary',
				paint: {
					'line-color': '#cc3700',
					'line-width': 2
				}
			});
		});

		return () => {
			map?.remove();
		};
	});
</script>

<main class="h-screen w-screen" bind:this={mapContainer} />
