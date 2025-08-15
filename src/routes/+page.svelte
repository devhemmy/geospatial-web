<script lang="ts">
	import { PUBLIC_MAPTILER_API_KEY } from '$env/static/public';
	import { onMount } from 'svelte';
	import maplibregl, { Map } from 'maplibre-gl';
	import 'maplibre-gl/dist/maplibre-gl.css';

	let mapContainer: HTMLElement;
	let map: Map;

	let isLoading = false;
	let isPanelOpen = true;
	let activeTab = 'layers';

	let showSatelliteLayer = true;
	let showAustriaBoundary = true;
	let satelliteOpacity = 100;
	let boundaryOpacity = 100;

	const basemaps = [
		{ id: 'streets', name: 'Streets', url: 'streets-v2' },
		{ id: 'satellite', name: 'Satellite', url: 'hybrid' },
		{ id: 'dark', name: 'Dark', url: 'streets-v2-dark' },
		{ id: 'light', name: 'Light', url: 'streets-v2-light' },
		{ id: 'terrain', name: 'Terrain', url: 'outdoor-v2' }
	];
	let currentBasemap = 'streets';

	let currentImageInfo = {
		date: '',
		cloudCover: 0,
		id: '',
		bbox: null as null | number[],
		collection: ''
	};

	let currentZoom = 6.5;
	let currentCenter = { lng: 14.55, lat: 47.52 };
	let mouseCoords = { lng: 0, lat: 0 };

	onMount(async () => {
		const apiKey = PUBLIC_MAPTILER_API_KEY;

		map = new Map({
			container: mapContainer,
			style: `https://api.maptiler.com/maps/streets-v2/style.json?key=${apiKey}`,
			center: [currentCenter.lng, currentCenter.lat],
			zoom: currentZoom
		});

		map.addControl(new maplibregl.NavigationControl(), 'bottom-right');
		map.addControl(new maplibregl.ScaleControl(), 'bottom-left');

		map.on('move', () => {
			const center = map.getCenter();
			currentCenter = { lng: center.lng, lat: center.lat };
			currentZoom = map.getZoom();
		});

		map.on('mousemove', (e) => {
			mouseCoords = {
				lng: parseFloat(e.lngLat.lng.toFixed(4)),
				lat: parseFloat(e.lngLat.lat.toFixed(4))
			};
		});

		map.on('load', async () => {
			map.addSource('austria-boundary', { type: 'geojson', data: '/austria.geojson' });
			map.addLayer({
				id: 'austria-outline',
				type: 'line',
				source: 'austria-boundary',
				paint: {
					'line-color': '#804000',
					'line-width': 2,
					'line-opacity': boundaryOpacity / 100
				}
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
			map.addLayer({
				id: 'cog-layer',
				type: 'raster',
				source: 'cog-source',
				paint: { 'raster-opacity': satelliteOpacity / 100 }
			});

			currentImageInfo = {
				date: 'Demo Image',
				cloudCover: 0,
				id: 'demo-cog',
				bbox: bbox,
				collection: 'Demo'
			};
		});

		return () => {
			map?.remove();
		};
	});

	function toggleSatelliteLayer() {
		if (map && map.getLayer('cog-layer')) {
			map.setLayoutProperty('cog-layer', 'visibility', showSatelliteLayer ? 'visible' : 'none');
		}
	}

	function toggleAustriaBoundary() {
		if (map && map.getLayer('austria-outline')) {
			map.setLayoutProperty(
				'austria-outline',
				'visibility',
				showAustriaBoundary ? 'visible' : 'none'
			);
		}
	}

	function updateSatelliteOpacity() {
		if (map && map.getLayer('cog-layer')) {
			map.setPaintProperty('cog-layer', 'raster-opacity', satelliteOpacity / 100);
		}
	}

	function updateBoundaryOpacity() {
		if (map && map.getLayer('austria-outline')) {
			map.setPaintProperty('austria-outline', 'line-opacity', boundaryOpacity / 100);
		}
	}

	async function switchBasemap(basemapId: string) {
		if (!map) return;
		currentBasemap = basemapId;
		const basemap = basemaps.find((b) => b.id === basemapId);
		if (basemap) {
			const apiKey = PUBLIC_MAPTILER_API_KEY;
			map.setStyle(`https://api.maptiler.com/maps/${basemap.url}/style.json?key=${apiKey}`);

			map.once('styledata', () => {
				if (!map.getSource('austria-boundary')) {
					map.addSource('austria-boundary', { type: 'geojson', data: '/austria.geojson' });
				}
				if (!map.getLayer('austria-outline')) {
					map.addLayer({
						id: 'austria-outline',
						type: 'line',
						source: 'austria-boundary',
						paint: {
							'line-color': '#804000',
							'line-width': 2,
							'line-opacity': boundaryOpacity / 100
						}
					});
				}

				if (!showAustriaBoundary) {
					map.setLayoutProperty('austria-outline', 'visibility', 'none');
				}
			});
		}
	}

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

			const imageDate = new Date(unsignedItem.properties.datetime);
			currentImageInfo = {
				date: imageDate.toLocaleDateString(),
				cloudCover: unsignedItem.properties['eo:cloud_cover'].toFixed(1),
				id: unsignedItem.id,
				bbox: unsignedItem.bbox,
				collection: unsignedItem.collection
			};

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
			map.addLayer({
				id: 'cog-layer',
				type: 'raster',
				source: 'cog-source',
				paint: { 'raster-opacity': satelliteOpacity / 100 }
			});

			if (!showSatelliteLayer) {
				map.setLayoutProperty('cog-layer', 'visibility', 'none');
			}
		} catch (error) {
			console.error('Failed to find or display new image:', error);
			alert('Could not find a new image. Please try again.');
		} finally {
			isLoading = false;
		}
	}

	function resetView() {
		if (map) {
			map.flyTo({
				center: [14.55, 47.52],
				zoom: 6.5,
				duration: 2000
			});
		}
	}

	function zoomToImage() {
		if (map && currentImageInfo.bbox) {
			const bounds: [[number, number], [number, number]] = [
				[currentImageInfo.bbox[0], currentImageInfo.bbox[1]],
				[currentImageInfo.bbox[2], currentImageInfo.bbox[3]]
			];
			map.fitBounds(bounds, { padding: 50, duration: 2000 });
		}
	}
</script>

<main class="relative h-screen w-screen bg-gray-50">
	<div class="h-full w-full" bind:this={mapContainer} />

	<button
		on:click={() => (isPanelOpen = !isPanelOpen)}
		class="absolute top-4 left-4 z-20 rounded-md bg-white px-3 py-2 shadow-lg transition-colors hover:bg-gray-100"
		aria-label="Toggle control panel"
	>
		Toggle
	</button>

	{#if isPanelOpen}
		<div class="absolute top-16 left-4 z-10 w-80 overflow-hidden rounded-lg bg-white shadow-xl">
			<div class="flex border-b bg-gray-50">
				<button
					on:click={() => (activeTab = 'layers')}
					class="flex-1 px-4 py-2 text-sm font-medium transition-colors {activeTab === 'layers'
						? 'border-b-2 border-blue-600 bg-white text-blue-600'
						: 'text-gray-600 hover:text-gray-900'}"
				>
					Layers
				</button>
				<button
					on:click={() => (activeTab = 'basemap')}
					class="flex-1 px-4 py-2 text-sm font-medium transition-colors {activeTab === 'basemap'
						? 'border-b-2 border-blue-600 bg-white text-blue-600'
						: 'text-gray-600 hover:text-gray-900'}"
				>
					Basemap
				</button>
				<button
					on:click={() => (activeTab = 'info')}
					class="flex-1 px-4 py-2 text-sm font-medium transition-colors {activeTab === 'info'
						? 'border-b-2 border-blue-600 bg-white text-blue-600'
						: 'text-gray-600 hover:text-gray-900'}"
				>
					Info
				</button>
			</div>

			<div
				class="scrollbar-thin scrollbar-track-gray-100 scrollbar-thumb-gray-400 hover:scrollbar-thumb-gray-500 max-h-96 overflow-y-auto p-4"
			>
				{#if activeTab === 'layers'}
					<div class="space-y-4">
						<div class="rounded-lg border bg-gray-50 p-3">
							<div class="mb-2 flex items-center justify-between">
								<label class="flex cursor-pointer items-center">
									<input
										type="checkbox"
										bind:checked={showSatelliteLayer}
										on:change={toggleSatelliteLayer}
										class="mr-2 h-4 w-4 rounded border-gray-300 text-blue-600 focus:ring-blue-500"
									/>
									<span class="font-medium">Satellite Imagery</span>
								</label>
							</div>
							{#if showSatelliteLayer}
								<div class="mt-2">
									<label class="text-xs text-gray-600">
										Opacity: {satelliteOpacity}%
										<input
											type="range"
											min="0"
											max="100"
											bind:value={satelliteOpacity}
											on:input={updateSatelliteOpacity}
											class="mt-1 w-full cursor-pointer appearance-none rounded-lg bg-gray-200 accent-blue-600"
										/>
									</label>
								</div>
							{/if}
						</div>

						<div class="rounded-lg border bg-gray-50 p-3">
							<div class="mb-2 flex items-center justify-between">
								<label class="flex cursor-pointer items-center">
									<input
										type="checkbox"
										bind:checked={showAustriaBoundary}
										on:change={toggleAustriaBoundary}
										class="mr-2 h-4 w-4 rounded border-gray-300 text-blue-600 focus:ring-blue-500"
									/>
									<span class="font-medium">Austria Boundary</span>
								</label>
							</div>
							{#if showAustriaBoundary}
								<div class="mt-2">
									<label class="text-xs text-gray-600">
										Opacity: {boundaryOpacity}%
										<input
											type="range"
											min="0"
											max="100"
											bind:value={boundaryOpacity}
											on:input={updateBoundaryOpacity}
											class="mt-1 w-full cursor-pointer appearance-none rounded-lg bg-gray-200 accent-blue-600"
										/>
									</label>
								</div>
							{/if}
						</div>
					</div>
				{/if}

				{#if activeTab === 'basemap'}
					<div class="space-y-2">
						{#each basemaps as basemap}
							<button
								on:click={() => switchBasemap(basemap.id)}
								class="w-full rounded-md px-3 py-2 text-left transition-colors {currentBasemap ===
								basemap.id
									? 'bg-blue-100 font-medium text-blue-700'
									: 'hover:bg-gray-100'}"
							>
								{basemap.name}
							</button>
						{/each}
					</div>
				{/if}

				{#if activeTab === 'info'}
					<div class="space-y-3">
						<div class="border-b pb-3">
							<h3 class="mb-2 font-medium text-gray-900">Current Image</h3>
							<div class="space-y-1 text-sm">
								<div>
									<span class="text-gray-600">Date:</span>
									<span class="font-medium">{currentImageInfo.date}</span>
								</div>
								<div>
									<span class="text-gray-600">Cloud Cover:</span>
									<span class="font-medium">{currentImageInfo.cloudCover}%</span>
								</div>
								<div>
									<span class="text-gray-600">Collection:</span>
									<span class="font-medium">{currentImageInfo.collection}</span>
								</div>
								<div class="mt-1 text-xs break-all text-gray-500">ID: {currentImageInfo.id}</div>
							</div>
							{#if currentImageInfo.bbox}
								<button
									on:click={zoomToImage}
									class="mt-2 rounded bg-blue-100 px-2 py-1 text-xs text-blue-700 transition-colors hover:bg-blue-200"
								>
									Zoom to Image
								</button>
							{/if}
						</div>

						<div class="border-b pb-3">
							<h3 class="mb-2 font-medium text-gray-900">Map View</h3>
							<div class="space-y-1 text-sm">
								<div>
									<span class="text-gray-600">Zoom:</span>
									<span class="font-medium">{currentZoom.toFixed(2)}</span>
								</div>
								<div>
									<span class="text-gray-600">Center:</span>
									<span class="font-medium"
										>{currentCenter.lat.toFixed(4)}, {currentCenter.lng.toFixed(4)}</span
									>
								</div>
								<div>
									<span class="text-gray-600">Cursor:</span>
									<span class="font-medium">{mouseCoords.lat}, {mouseCoords.lng}</span>
								</div>
							</div>
							<button
								on:click={resetView}
								class="mt-2 rounded bg-gray-100 px-2 py-1 text-xs text-gray-700 transition-colors hover:bg-gray-200"
							>
								Reset View
							</button>
						</div>
					</div>
				{/if}
			</div>
		</div>
	{/if}

	<div class="absolute top-4 right-4 z-10">
		<button
			on:click={findAndDisplayNewImage}
			disabled={isLoading}
			class="rounded-md bg-blue-600 px-4 py-2 font-bold text-white shadow-lg transition-colors hover:bg-blue-700 disabled:bg-gray-400"
		>
			{#if isLoading}
				<span class="flex items-center"> Searching... </span>
			{:else}
				Find New Satellite Image
			{/if}
		</button>
	</div>

	<div
		class="absolute right-4 bottom-12 z-10 rounded bg-white px-3 py-1 font-mono text-xs shadow-md"
	>
		Lat: {mouseCoords.lat} | Lng: {mouseCoords.lng}
	</div>
</main>
