<script lang="ts">
	import { createEventDispatcher, onMount } from 'svelte';
	import { MapLibre, Marker, MapEvents } from 'svelte-maplibre';
	import { number, t } from 'svelte-i18n';
	import { getBasemapUrl } from '$lib';
	import CategoryDropdown from '../CategoryDropdown.svelte';
	import type { Collection, Location } from '$lib/types';

	// Icons
	import SearchIcon from '~icons/mdi/magnify';
	import LocationIcon from '~icons/mdi/crosshairs-gps';
	import MapIcon from '~icons/mdi/map';
	import CheckIcon from '~icons/mdi/check';
	import ClearIcon from '~icons/mdi/close';
	import PinIcon from '~icons/mdi/map-marker';
	import InfoIcon from '~icons/mdi/information';
	import StarIcon from '~icons/mdi/star';
	import LinkIcon from '~icons/mdi/link';
	import TextIcon from '~icons/mdi/text';
	import CategoryIcon from '~icons/mdi/tag';
	import PublicIcon from '~icons/mdi/earth';
	import GenerateIcon from '~icons/mdi/lightning-bolt';
	import ArrowLeftIcon from '~icons/mdi/arrow-left';
	import SaveIcon from '~icons/mdi/content-save';
	import type { Category, User } from '$lib/types';
	import MarkdownEditor from '../MarkdownEditor.svelte';
	import TagComplete from '../TagComplete.svelte';

	const dispatch = createEventDispatcher();

	// Location selection properties
	let searchQuery = '';
	let searchResults: any[] = [];
	let selectedLocation: any = null;
	let mapCenter: [number, number] = [-74.5, 40];
	let mapZoom = 2;
	let isSearching = false;
	let isReverseGeocoding = false;
	let searchTimeout: ReturnType<typeof setTimeout>;
	let mapComponent: any;
	let selectedMarker: { lng: number; lat: number } | null = null;

	// Enhanced location data
	let locationData: {
		city?: { name: string; id: string; visited: boolean };
		region?: { name: string; id: string; visited: boolean };
		country?: { name: string; country_code: string; visited: boolean };
		display_name?: string;
		location_name?: string;
	} | null = null;

	// Form data properties
	let location: {
		name: string;
		category: Category | null;
		rating: number;
		is_public: boolean;
		link: string;
		description: string;
		latitude: number | null;
		longitude: number | null;
		location: string;
		tags: string[];
		collections?: string[];
	} = {
		name: '',
		category: null,
		rating: NaN,
		is_public: false,
		link: '',
		description: '',
		latitude: null,
		longitude: null,
		location: '',
		tags: [],
		collections: []
	};

	let user: User | null = null;
	let locationToEdit: Location | null = null;
	let wikiError = '';
	let isGeneratingDesc = false;
	let ownerUser: User | null = null;

	// Props (would be passed in from parent component)
	export let initialLocation: any = null;
	export let currentUser: any = null;
	export let editingLocation: any = null;
	export let collection: Collection | null = null;

	$: user = currentUser;
	$: locationToEdit = editingLocation;

	// Location selection functions
	async function searchLocations(query: string) {
		if (!query.trim() || query.length < 3) {
			searchResults = [];
			return;
		}

		isSearching = true;
		try {
			const response = await fetch(
				`/api/reverse-geocode/search/?query=${encodeURIComponent(query)}`
			);
			const results = await response.json();

			searchResults = results.map((result: any) => ({
				id: result.name + result.lat + result.lon,
				name: result.name,
				lat: parseFloat(result.lat),
				lng: parseFloat(result.lon),
				type: result.type,
				category: result.category,
				location: result.display_name,
				importance: result.importance,
				powered_by: result.powered_by
			}));
		} catch (error) {
			console.error('Search error:', error);
			searchResults = [];
		} finally {
			isSearching = false;
		}
	}

	function handleSearchInput() {
		clearTimeout(searchTimeout);
		searchTimeout = setTimeout(() => {
			searchLocations(searchQuery);
		}, 300);
	}

	async function selectSearchResult(searchResult: any) {
		selectedLocation = searchResult;
		selectedMarker = { lng: searchResult.lng, lat: searchResult.lat };
		mapCenter = [searchResult.lng, searchResult.lat];
		mapZoom = 14;
		searchResults = [];
		searchQuery = searchResult.name;

		// Update form data
		if (!location.name) location.name = searchResult.name;
		location.latitude = searchResult.lat;
		location.longitude = searchResult.lng;
		location.name = searchResult.name;

		await performDetailedReverseGeocode(searchResult.lat, searchResult.lng);
	}

	async function handleMapClick(e: { detail: { lngLat: { lng: number; lat: number } } }) {
		selectedMarker = {
			lng: e.detail.lngLat.lng,
			lat: e.detail.lngLat.lat
		};

		await reverseGeocode(e.detail.lngLat.lng, e.detail.lngLat.lat);
	}

	async function reverseGeocode(lng: number, lat: number) {
		isReverseGeocoding = true;

		try {
			const response = await fetch(`/api/reverse-geocode/search/?query=${lat},${lng}`);
			const results = await response.json();

			if (results && results.length > 0) {
				const result = results[0];
				selectedLocation = {
					name: result.name,
					lat: lat,
					lng: lng,
					location: result.display_name,
					type: result.type,
					category: result.category
				};
				searchQuery = result.name;
				if (!location.name) location.name = result.name;
			} else {
				selectedLocation = {
					name: `Location at ${lat.toFixed(4)}, ${lng.toFixed(4)}`,
					lat: lat,
					lng: lng,
					location: `${lat.toFixed(4)}, ${lng.toFixed(4)}`
				};
				searchQuery = selectedLocation.name;
				if (!location.name) location.name = selectedLocation.name;
			}

			location.latitude = lat;
			location.longitude = lng;
			location.location = selectedLocation.location;

			await performDetailedReverseGeocode(lat, lng);
		} catch (error) {
			console.error('Reverse geocoding error:', error);
			selectedLocation = {
				name: `Location at ${lat.toFixed(4)}, ${lng.toFixed(4)}`,
				lat: lat,
				lng: lng,
				location: `${lat.toFixed(4)}, ${lng.toFixed(4)}`
			};
			searchQuery = selectedLocation.name;
			if (!location.name) location.name = selectedLocation.name;
			location.latitude = lat;
			location.longitude = lng;
			location.location = selectedLocation.location;
			locationData = null;
		} finally {
			isReverseGeocoding = false;
		}
	}

	async function performDetailedReverseGeocode(lat: number, lng: number) {
		try {
			const response = await fetch(
				`/api/reverse-geocode/reverse_geocode/?lat=${lat}&lon=${lng}&format=json`
			);

			if (response.ok) {
				const data = await response.json();
				locationData = {
					city: data.city
						? {
								name: data.city,
								id: data.city_id,
								visited: data.city_visited || false
							}
						: undefined,
					region: data.region
						? {
								name: data.region,
								id: data.region_id,
								visited: data.region_visited || false
							}
						: undefined,
					country: data.country
						? {
								name: data.country,
								country_code: data.country_id,
								visited: false
							}
						: undefined,
					display_name: data.display_name,
					location_name: data.location_name
				};
				location.location = data.display_name;
			} else {
				locationData = null;
			}
		} catch (error) {
			console.error('Detailed reverse geocoding error:', error);
			locationData = null;
		}
	}

	function useCurrentLocation() {
		if ('geolocation' in navigator) {
			navigator.geolocation.getCurrentPosition(
				async (position) => {
					const lat = position.coords.latitude;
					const lng = position.coords.longitude;
					selectedMarker = { lng, lat };
					mapCenter = [lng, lat];
					mapZoom = 14;
					await reverseGeocode(lng, lat);
				},
				(error) => {
					console.error('Geolocation error:', error);
				}
			);
		}
	}

	function clearLocationSelection() {
		selectedLocation = null;
		selectedMarker = null;
		locationData = null;
		searchQuery = '';
		searchResults = [];
		location.latitude = null;
		location.longitude = null;
		location.location = '';
		mapCenter = [-74.5, 40];
		mapZoom = 2;
	}

	async function generateDesc() {
		if (!location.name) return;

		isGeneratingDesc = true;
		wikiError = '';

		try {
			// Mock Wikipedia API call - replace with actual implementation
			const response = await fetch(`/api/generate/desc/?name=${encodeURIComponent(location.name)}`);
			if (response.ok) {
				const data = await response.json();
				location.description = data.extract || '';
			} else {
				wikiError = `${$t('adventures.wikipedia_error') || 'Error fetching description from Wikipedia'}`;
			}
		} catch (error) {
			wikiError = `${$t('adventures.wikipedia_error') || ''}`;
		} finally {
			isGeneratingDesc = false;
		}
	}

	async function handleSave() {
		if (!location.name || !location.category) {
			return;
		}

		// round latitude and longitude to 6 decimal places
		if (location.latitude !== null && typeof location.latitude === 'number') {
			location.latitude = parseFloat(location.latitude.toFixed(6));
		}
		if (location.longitude !== null && typeof location.longitude === 'number') {
			location.longitude = parseFloat(location.longitude.toFixed(6));
		}
		if (collection && collection.id) {
			location.collections = [collection.id];
		}

		// either a post or a patch depending on whether we're editing or creating
		if (locationToEdit && locationToEdit.id) {
			let res = await fetch(`/api/locations/${locationToEdit.id}`, {
				method: 'PATCH',
				headers: {
					'Content-Type': 'application/json'
				},
				body: JSON.stringify(location)
			});
			let updatedLocation = await res.json();
			location = updatedLocation;
		} else {
			let res = await fetch(`/api/locations`, {
				method: 'POST',
				headers: {
					'Content-Type': 'application/json'
				},
				body: JSON.stringify(location)
			});
			let newLocation = await res.json();
			location = newLocation;
		}

		dispatch('save', {
			...location
		});
	}

	function handleBack() {
		dispatch('back');
	}

	onMount(async () => {
		if (initialLocation.latitude && initialLocation.longitude) {
			selectedMarker = {
				lng: initialLocation.longitude,
				lat: initialLocation.latitude
			};
			location.latitude = initialLocation.latitude;
			location.longitude = initialLocation.longitude;
			mapCenter = [initialLocation.longitude, initialLocation.latitude];
			mapZoom = 14;
			selectedLocation = {
				name: initialLocation.name || '',
				lat: initialLocation.latitude,
				lng: initialLocation.longitude,
				location: initialLocation.location || '',
				type: 'point',
				category: initialLocation.category || null
			};
			selectedMarker = {
				lng: Number(initialLocation.longitude),
				lat: Number(initialLocation.latitude)
			};
			// trigger reverse geocoding to populate location data
			await performDetailedReverseGeocode(initialLocation.latitude, initialLocation.longitude);
		}
	});

	onMount(() => {
		if (initialLocation && typeof initialLocation === 'object') {
			// Only update location properties if they don't already have values
			// This prevents overwriting user selections
			if (!location.name) location.name = initialLocation.name || '';
			if (!location.link) location.link = initialLocation.link || '';
			if (!location.description) location.description = initialLocation.description || '';
			if (Number.isNaN(location.rating)) location.rating = initialLocation.rating || NaN;
			if (location.is_public === false) location.is_public = initialLocation.is_public || false;

			// Only set category if location doesn't have one or if initialLocation has a valid category
			if (!location.category || !location.category.id) {
				if (initialLocation.category && initialLocation.category.id) {
					location.category = initialLocation.category;
				}
			}

			if (initialLocation.tags && Array.isArray(initialLocation.tags)) {
				location.tags = initialLocation.tags;
			}

			if (initialLocation.location) {
				location.location = initialLocation.location;
			}

			if (initialLocation.user) {
				ownerUser = initialLocation.user;
			}
		}

		searchQuery = initialLocation.name || '';
		return () => {
			clearTimeout(searchTimeout);
		};
	});
</script>

<div class="bg-gradient-to-br from-base-200/30 via-base-100 to-primary/5 p-6">
	<div class="max-w-full mx-auto space-y-6">
		<!-- Basic Information Section -->
		<div class="card bg-base-100 border border-base-300 shadow-lg">
			<div class="card-body p-6">
				<div class="flex items-center gap-3 mb-6">
					<div class="p-2 bg-primary/10 rounded-lg">
						<InfoIcon class="w-5 h-5 text-primary" />
					</div>
					<h2 class="text-xl font-bold">{$t('adventures.basic_information')}</h2>
				</div>

				<div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
					<!-- Left Column -->
					<div class="space-y-4">
						<!-- Name Field -->
						<div class="form-control">
							<label class="label" for="name">
								<span class="label-text font-medium">
									{$t('adventures.name')} <span class="text-error">*</span>
								</span>
							</label>
							<input
								type="text"
								id="name"
								bind:value={location.name}
								class="input input-bordered bg-base-100/80 focus:bg-base-100"
								placeholder="Enter location name"
								required
							/>
						</div>

						<!-- Category Field -->
						<div class="form-control">
							<label class="label" for="category">
								<span class="label-text font-medium">
									{$t('adventures.category')} <span class="text-error">*</span>
								</span>
							</label>
							{#if (user && ownerUser && user.uuid == ownerUser.uuid) || !ownerUser}
								<CategoryDropdown bind:selected_category={location.category} />
							{:else}
								<div
									class="flex items-center gap-3 p-3 bg-base-100/80 border border-base-300 rounded-lg"
								>
									{#if location.category?.icon}
										<span class="text-xl flex-shrink-0">{location.category.icon}</span>
									{/if}
									<span class="font-medium">
										{location.category?.display_name || location.category?.name}
									</span>
								</div>
							{/if}
						</div>
					</div>

					<!-- Right Column -->
					<div class="space-y-4">
						<!-- Link Field -->
						<div class="form-control">
							<label class="label" for="link">
								<span class="label-text font-medium">{$t('adventures.link')}</span>
							</label>
							<input
								type="url"
								id="link"
								bind:value={location.link}
								class="input input-bordered bg-base-100/80 focus:bg-base-100"
								placeholder="https://example.com"
							/>
						</div>

						<!-- Description Field -->
						<div class="form-control">
							<label class="label" for="description">
								<span class="label-text font-medium">{$t('adventures.description')}</span>
							</label>
							<MarkdownEditor bind:text={location.description} editor_height="h-32" />

							<div class="flex items-center gap-4 mt-3">
								<button
									type="button"
									class="btn btn-neutral btn-sm gap-2"
									on:click={generateDesc}
									disabled={!location.name || isGeneratingDesc}
								>
									{#if isGeneratingDesc}
										<span class="loading loading-spinner loading-xs"></span>
									{:else}
										<GenerateIcon class="w-4 h-4" />
									{/if}
									{$t('adventures.generate_desc')}
								</button>
								{#if wikiError}
									<div class="alert alert-error alert-sm">
										<InfoIcon class="w-4 h-4" />
										<span class="text-sm">{wikiError}</span>
									</div>
								{/if}
							</div>
						</div>
					</div>
				</div>
			</div>
		</div>

		<!-- Action Buttons -->
		<div class="flex gap-3 justify-end pt-4">
			<button class="btn btn-neutral-200 gap-2" on:click={handleBack}>
				<ArrowLeftIcon class="w-5 h-5" />
				{$t('adventures.back')}
			</button>
			<button
				class="btn btn-primary gap-2"
				disabled={!location.name || !location.category || isReverseGeocoding}
				on:click={handleSave}
			>
				{#if isReverseGeocoding}
					<span class="loading loading-spinner loading-sm"></span>
					{$t('adventures.processing')}...
				{:else}
					<SaveIcon class="w-5 h-5" />
					{$t('adventures.continue')}
				{/if}
			</button>
		</div>
	</div>
</div>
