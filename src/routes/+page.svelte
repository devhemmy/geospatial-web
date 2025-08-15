<script lang="ts">
	import { PUBLIC_MAPTILER_API_KEY } from '$env/static/public';
	import { onMount } from 'svelte';
	import maplibregl, { Map } from 'maplibre-gl';
	import 'maplibre-gl/dist/maplibre-gl.css';

	let mapContainer: HTMLElement;
	let map: Map;

	let isLoading = false;

	onMount(async () => {
		const apiKey = PUBLIC_MAPTILER_API_KEY;

		map = new Map({
			container: mapContainer,
			style: `https://api.maptiler.com/maps/streets-v2/style.json?key=${apiKey}`,
			center: [14.55, 47.52],
			zoom: 6.5
		});

		map.on('load', async () => {
			map.addSource('austria-boundary', { type: 'geojson', data: '/austria.geojson' });
			map.addLayer({
				id: 'austria-outline',
				type: 'line',
				source: 'austria-boundary',
				paint: { 'line-color': '#804000', 'line-width': 2 }
			});

			const initialCogUrl = 'https://maplibre.org/maplibre-gl-js/docs/assets/cog.tif';
			const initialTilejsonUrl = `https://titiler.xyz/cog/WebMercatorQuad/tilejson.json?url=${encodeURIComponent(initialCogUrl)}&bidx=1&bidx=2&bidx=3`;

			const response = await fetch(initialTilejsonUrl);
			const tileJson = await response.json();

			const bbox = tileJson.bounds;
			const bounds: [[number, number], [number, number]] = [
				[bbox[0], bbox[1]],
				[bbox[2], bbox[3]]
			];

			map.fitBounds(bounds, { padding: 50, duration: 3000, maxZoom: 14 });

			map.addSource('cog-source', { type: 'raster', url: initialTilejsonUrl, tileSize: 256 });
			map.addLayer({ id: 'cog-layer', type: 'raster', source: 'cog-source' });
		});

		return () => {
			map?.remove();
		};
	});

	async function findAndDisplayNewImage() {
		isLoading = true;
		try {
			const stacApiUrl = 'https://planetarycomputer.microsoft.com/api/stac/v1/search';
			const searchQuery = {
				limit: 50,
				'filter-lang': 'cql2-json',
				filter: {
					op: 'and',
					args: [
						{ op: '=', args: [{ property: 'collection' }, 'sentinel-2-l2a'] },
						{
							op: 's_intersects',
							args: [{ property: 'geometry' }, { type: 'Point', coordinates: [13.33, 47.73] }]
						},
						{ op: '<=', args: [{ property: 'properties.eo:cloud_cover' }, 10] }
					]
				},
				sortby: [{ field: 'properties.datetime', direction: 'desc' }]
			};
			const stacResponse = await fetch(stacApiUrl, {
				method: 'POST',
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify(searchQuery)
			});
			const stacData = await stacResponse.json();
			const randomIndex = Math.floor(Math.random() * stacData.features.length);
			const unsignedItem = stacData.features[randomIndex];
			const unsignedCogUrl = unsignedItem.assets.visual.href;

			const signingApiUrl = `https://planetarycomputer.microsoft.com/api/sas/v1/sign?href=${encodeURIComponent(unsignedCogUrl)}`;
			const signedResponse = await fetch(signingApiUrl);
			const signedData = await signedResponse.json();
			const signedCogUrl = signedData.href;

			const newTilejsonUrl = `https://titiler.xyz/cog/WebMercatorQuad/tilejson.json?url=${encodeURIComponent(signedCogUrl)}&bidx=1&bidx=2&bidx=3`;
			const tilejsonResponse = await fetch(newTilejsonUrl);
			const tileJson = await tilejsonResponse.json();

			const bbox = tileJson.bounds;
			const newBounds: [[number, number], [number, number]] = [
				[bbox[0], bbox[1]],
				[bbox[2], bbox[3]]
			];
			map.fitBounds(newBounds, {
				padding: 50,
				duration: 2000,
				maxZoom: 12
			});

			if (map.getLayer('cog-layer')) map.removeLayer('cog-layer');
			if (map.getSource('cog-source')) map.removeSource('cog-source');

			map.addSource('cog-source', { type: 'raster', url: newTilejsonUrl, tileSize: 256 });
			map.addLayer({ id: 'cog-layer', type: 'raster', source: 'cog-source' });
		} catch (error) {
			console.error('Failed to find or display new image:', error);
			alert('Could not find a new image. Please try again.');
		} finally {
			isLoading = false;
		}
	}
</script>

<main class="relative h-screen w-screen">
	<div class="h-full w-full" bind:this={mapContainer} />
	<div class="absolute top-4 right-4 z-10">
		<button
			on:click={findAndDisplayNewImage}
			disabled={isLoading}
			class="rounded-md bg-blue-600 px-4 py-2 font-bold text-white shadow-lg transition-colors hover:bg-blue-700 disabled:bg-gray-400"
		>
			{#if isLoading}
				Searching...
			{:else}
				Find New Satellite Image
			{/if}
		</button>
	</div>
</main>
