<script>
	import { onDestroy, onMount } from 'svelte';
	import { base } from '$app/paths';
	import maplibregl from 'maplibre-gl';
	import { supabase } from '$lib/supabaseClient';

	const STYLES = {
		dark: 'https://basemaps.cartocdn.com/gl/dark-matter-gl-style/style.json',
		light: 'https://basemaps.cartocdn.com/gl/positron-gl-style/style.json'
	};

	const BCOLORS = {
		dark: { lo: '#bcd8f2', mid: '#5da8e0', hi: '#2564a8', top: '#133560' },
		light: { lo: '#ffe4c8', mid: '#ffb468', hi: '#e05f10', top: '#962800' }
	};

	const TERRAIN_EXAG = 1.5;
	const BUILDING_OPACITY = 0.9;

	const CAMPS = {
		e: {
			label: 'Edinburg Campus',
			center: [-98.17383, 26.30621],
			zoom: 15.5,
			pitch: 60,
			bearing: -20,
			addr: '1201 W. University Dr., Edinburg, TX 78539',
			phone: '(956) 665-7338',
			enroll: '~24,000 students',
			colls: 'Engineering, Business, Sciences, Education, Fine Arts & more'
		},
		b: {
			label: 'Brownsville Campus',
			center: [-97.48716, 25.8951],
			zoom: 15.5,
			pitch: 60,
			bearing: 20,
			addr: 'One W. University Blvd., Brownsville, TX 78520',
			phone: '(956) 882-7624',
			enroll: '~10,000 students',
			colls: 'Health Sciences, Liberal Arts, STEM, Education & more'
		}
	};

	const BUILDINGS = {
		e: [
			{ code: 'ESSBL', name: 'Student Services Building', type: 'Student Services', coords: [-98.17422, 26.30467] },
			{ code: 'ELIBR', name: 'University Library', type: 'Library', coords: [-98.17383, 26.30678] },
			{
				code: 'ECOBE',
				name: 'Robert C. Vackar College of Business',
				type: 'Business',
				coords: [-98.17342, 26.30754]
			},
			{ code: 'ESCNE', name: 'Science Building', type: 'Sciences', coords: [-98.17219, 26.30654] },
			{ code: 'EEDUC', name: 'Education Complex', type: 'Education', coords: [-98.17159, 26.3083] },
			{ code: 'ECCTR', name: 'Computer Center', type: 'Computing / IT', coords: [-98.17301, 26.30803] },
			{ code: 'EMAGC', name: 'Mathematics & General Classrooms', type: 'Academic', coords: [-98.17292, 26.30762] },
			{ code: 'EPLAN', name: 'H.E.B. Planetarium', type: 'Planetarium', coords: [-98.17193, 26.30648] },
			{ code: 'EPACC', name: 'Performing Arts Complex C', type: 'Performing Arts', coords: [-98.173, 26.30427] },
			{ code: 'EPACB', name: 'Performing Arts Complex B', type: 'Performing Arts', coords: [-98.17317, 26.30379] },
			{ code: 'EINNV', name: 'Innovation Building', type: 'Research', coords: [-98.17388, 26.30574] },
			{ code: 'EACSB', name: 'Academic Services Building', type: 'Academic Services', coords: [-98.17263, 26.30529] },
			{ code: 'EUREC', name: 'University Recreation Center', type: 'Recreation', coords: [-98.17775, 26.31015] },
			{ code: 'EHRTG', name: 'Heritage Hall', type: 'Academic', coords: [-98.17744, 26.3063] },
			{ code: 'ETROX', name: 'Troxel Hall', type: 'Academic', coords: [-98.17738, 26.30658] },
			{
				code: 'EIEAB',
				name: 'Interdisciplinary Engineering & Academic Bldg',
				type: 'Engineering',
				coords: [-98.17485, 26.30622]
			},
			{ code: 'EHPE1', name: 'Health & Physical Education Complex', type: 'Health / Recreation', coords: [-98.17025, 26.30561] },
			{ code: 'ELCTR', name: 'The Learning Center', type: 'Academic Support', coords: [-98.17306, 26.30635] },
			{ code: 'ELABS', name: 'Liberal Arts Building South', type: 'Liberal Arts', coords: [-98.17592, 26.3061] },
			{ code: 'EEMLH', name: 'Emilia Schunior Ramirez Hall', type: 'Academic', coords: [-98.17746, 26.30587] },
			{ code: 'ESWKH', name: 'Southwick Hall', type: 'Academic', coords: [-98.1727, 26.30469] },
			{
				code: 'ECESS',
				name: 'Community Engagement & Student Success',
				type: 'Student Services',
				coords: [-98.15076, 26.2873]
			}
		],
		b: [
			{ code: 'BMAIN', name: 'Main Building (The Tower)', type: 'Administration', coords: [-97.48681, 25.89363] },
			{ code: 'BLIBR', name: 'University Library', type: 'Library', coords: [-97.48792, 25.89319] },
			{
				code: 'BSETB',
				name: 'Science & Engineering Technology Building',
				type: 'Sciences / Eng.',
				coords: [-97.48848, 25.89723]
			},
			{ code: 'BLHSB', name: 'Life & Health Sciences Building', type: 'Health Sciences', coords: [-97.48701, 25.89639] },
			{
				code: 'BLCBR',
				name: 'Luis V. Colom Biomedical Research Facility',
				type: 'Research',
				coords: [-97.48575, 25.89561]
			},
			{ code: 'BSTUN', name: 'Student Union Building', type: 'Student Life', coords: [-97.48805, 25.896] }
		]
	};

	let theme = $state('dark');
	let terrainOn = true;
	let rotInt = null;
	let syncing = false;
	let activeCampus = 'e';
	let selectedBldg = null;
	let campusPopups = { dark: null, light: null };
	let selectedBuilding = $state(null);
	let showOfficial = $state(false);
	let showUnofficial = $state(false);
	let officialEvents = $state([]);
	let unofficialEvents = $state([]);
	let selectedEvent = $state(null);
	let ready = { dark: false, light: false };
	let blistOpen = false;

	let maps = null;
	let keyHandler = null;

	function camOf(m) {
		return { center: m.getCenter(), zoom: m.getZoom(), pitch: m.getPitch(), bearing: m.getBearing() };
	}

	function syncTo(src, tgt) {
		if (syncing) return;
		syncing = true;
		tgt.jumpTo(camOf(src));
		syncing = false;
	}

	function findBuildingSource(m) {
		return (
			(m.getStyle()?.layers ?? []).find((l) => l['source-layer'] === 'building' && l.type !== 'fill-extrusion')?.source ?? null
		);
	}

	function addBuildings(m, t) {
		if (m.getLayer('bld3d')) m.removeLayer('bld3d');
		const src = findBuildingSource(m);
		if (!src) return;
		const c = BCOLORS[t];
		const sym = (m.getStyle()?.layers ?? []).find((l) => l.type === 'symbol')?.id;
		m.addLayer(
			{
				id: 'bld3d',
				source: src,
				'source-layer': 'building',
				type: 'fill-extrusion',
				minzoom: 13,
				paint: {
					'fill-extrusion-color': [
						'interpolate',
						['linear'],
						['coalesce', ['get', 'render_height'], ['get', 'height'], 0],
						0,
						c.lo,
						20,
						c.mid,
						60,
						c.hi,
						150,
						c.top
					],
					'fill-extrusion-height': [
						'interpolate',
						['linear'],
						['zoom'],
						13,
						0,
						14,
						['coalesce', ['get', 'render_height'], ['get', 'height'], 5]
					],
					'fill-extrusion-base': [
						'interpolate',
						['linear'],
						['zoom'],
						13,
						0,
						14,
						['coalesce', ['get', 'render_min_height'], ['get', 'min_height'], 0]
					],
					'fill-extrusion-opacity': BUILDING_OPACITY
				}
			},
			sym
		);

		if (!document.getElementById('tog-b').checked) m.setLayoutProperty('bld3d', 'visibility', 'none');
	}

	function addCampusMarker(m, t, id, c) {
		const el = document.createElement('div');
		el.className = 'mrkr';
		el.id = `mk-${t}-${id}`;
		const popup = new maplibregl.Popup({ offset: 26, closeButton: true, closeOnClick: false, maxWidth: '260px' }).setHTML(
			`<div class="pname">${c.label}</div><div class="prow"><div class="plbl">Address</div><div class="pval">${c.addr}</div></div><div class="prow"><div class="plbl">Phone</div><div class="pval">${c.phone}</div></div><div class="prow"><div class="plbl">Enrollment</div><div class="pval">${c.enroll}</div></div><div class="prow"><div class="plbl">Colleges</div><div class="pval">${c.colls}</div></div>`
		);
		el.addEventListener('click', () => {
			if (campusPopups[t]) campusPopups[t].remove();
			popup.addTo(m);
			campusPopups[t] = popup;
			flyTo(id);
		});
		new maplibregl.Marker({ element: el, anchor: 'bottom' }).setLngLat(c.center).addTo(m);
	}

	function addBuildingLayers(m, t) {
		const pinColor = t === 'dark' ? 'rgba(96,165,250,0.92)' : 'rgba(255,107,0,0.92)';
		const selColor = '#ffffff';
		const strokeNorm = '#ffffff';
		const strokeSel = t === 'dark' ? '#60a5fa' : '#ff6b00';

		['e', 'b'].forEach((campus) => {
			const srcId = `bsrc-${campus}`;
			const lyrId = `bpins-${campus}`;

			m.addSource(srcId, {
				type: 'geojson',
				data: {
					type: 'FeatureCollection',
					features: BUILDINGS[campus].map((b) => ({
						type: 'Feature',
						id: b.code,
						geometry: { type: 'Point', coordinates: b.coords },
						properties: { code: b.code, name: b.name, type: b.type, campus }
					}))
				}
			});

			m.addLayer({
				id: lyrId,
				type: 'circle',
				source: srcId,
				paint: {
					'circle-radius': ['interpolate', ['linear'], ['zoom'], 10, 4, 15, 7, 18, 10],
					'circle-color': ['case', ['boolean', ['feature-state', 'selected'], false], selColor, pinColor],
					'circle-stroke-width': ['case', ['boolean', ['feature-state', 'selected'], false], 3, 2],
					'circle-stroke-color': ['case', ['boolean', ['feature-state', 'selected'], false], strokeSel, strokeNorm]
				}
			});

			m.on('mouseenter', lyrId, () => {
				m.getCanvas().style.cursor = 'pointer';
			});
			m.on('mouseleave', lyrId, () => {
				m.getCanvas().style.cursor = '';
			});

			m.on('click', lyrId, (e) => {
				e.preventDefault();
				const props = e.features[0].properties;
				const bldg = BUILDINGS[props.campus]?.find((b) => b.code === props.code);
				if (bldg) selectBuilding(bldg);
			});
		});
	}

	function renderBuildingList(campusId) {
		const list = document.getElementById('blist');
		list.innerHTML = '';
		BUILDINGS[campusId].forEach((b) => {
			const btn = document.createElement('button');
			btn.className = 'blitem';
			btn.id = `bli-${b.code}`;
			btn.innerHTML = `<span class="blcode">${b.code}</span><span class="blname">${b.name}</span>`;
			btn.addEventListener('click', () => selectBuilding(b));
			list.appendChild(btn);
		});
	}

	function toggleBuildingList() {
		blistOpen = !blistOpen;
		const list = document.getElementById('blist');
		const arrow = document.getElementById('barrow');
		list.classList.toggle('open', blistOpen);
		arrow.classList.toggle('open', blistOpen);
	}

	function selectBuilding(bldg) {
		const campus = Object.entries(BUILDINGS).find(([, arr]) => arr.some((b) => b.code === bldg.code))?.[0];
		if (!campus) return;
		selectedBuilding = { ...bldg, campus };

		if (selectedBldg) {
			['dark', 'light'].forEach((t) => {
				['e', 'b'].forEach((c) => {
					const src = `bsrc-${c}`;
					if (maps[t].getSource(src)) {
						maps[t].setFeatureState({ source: src, id: selectedBldg }, { selected: false });
					}
				});
			});
		}

		document.querySelectorAll('.blitem').forEach((b) => b.classList.remove('active'));

		selectedBldg = bldg.code;

		['dark', 'light'].forEach((t) => {
			const src = `bsrc-${campus}`;
			if (maps[t].getSource(src)) {
				maps[t].setFeatureState({ source: src, id: bldg.code }, { selected: true });
			}
		});

		document.getElementById(`bli-${bldg.code}`)?.classList.add('active');
		document.getElementById(`bli-${bldg.code}`)?.scrollIntoView({ block: 'nearest', behavior: 'smooth' });

		maps[theme].flyTo({
			center: bldg.coords,
			zoom: 17.5,
			pitch: 60,
			bearing: maps[theme].getBearing(),
			speed: 0.5,
			curve: 1.4,
			easing: (t) => (t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t)
		});
	}

	function toggleTheme() {
		const next = theme === 'dark' ? 'light' : 'dark';
		document.getElementById(`map-${theme}`).classList.remove('active');
		document.getElementById(`map-${next}`).classList.add('active');
		const activeId = document.querySelector('.cbtn.active')?.id?.replace('btn-', '');
		if (activeId) {
			document.querySelectorAll('.mrkr').forEach((m) => m.classList.remove('active'));
			document.getElementById(`mk-${next}-${activeId}`)?.classList.add('active');
		}
		theme = next;
		document.documentElement.setAttribute('data-theme', theme);
	}

	function showBadge(msg) {
		const el = document.getElementById('badge');
		el.textContent = msg;
		el.style.display = 'block';
	}

	function hideBadge() {
		document.getElementById('badge').style.display = 'none';
	}

	function clearSelectedBuilding() {
		if (selectedBldg) {
			['dark', 'light'].forEach((t) => {
				['e', 'b'].forEach((c) => {
					const src = `bsrc-${c}`;
					if (maps[t].getSource(src)) {
						maps[t].setFeatureState({ source: src, id: selectedBldg }, { selected: false });
					}
				});
			});
			selectedBldg = null;
		}
		selectedBuilding = null;
		clearSelectedEvent();
		document.querySelectorAll('.blitem').forEach((b) => b.classList.remove('active'));
	}

	function toggleOfficial() {
		showOfficial = !showOfficial;
		updateEventVisibility();
	}

	function toggleUnofficial() {
		showUnofficial = !showUnofficial;
		updateEventVisibility();
	}

	function getBuildingCoords(campus, code) {
		const building = BUILDINGS[campus]?.find((b) => b.code === code);
		return building?.coords ?? null;
	}

	function getEventCoords(event) {
		if (event?.campus && event?.building_code) return getBuildingCoords(event.campus, event.building_code);
		return null;
	}

	function addEventLayers(m, t) {
		const officialColor = t === 'dark' ? 'rgba(34,197,94,0.95)' : 'rgba(22,163,74,0.95)';
		const unofficialColor = t === 'dark' ? 'rgba(250,204,21,0.95)' : 'rgba(202,138,4,0.95)';

		const defs = [
			{ kind: 'official', src: 'esrc-official', layer: 'epins-official', color: officialColor },
			{ kind: 'unofficial', src: 'esrc-unofficial', layer: 'epins-unofficial', color: unofficialColor }
		];

		for (const d of defs) {
			if (!m.getSource(d.src)) {
				m.addSource(d.src, { type: 'geojson', data: { type: 'FeatureCollection', features: [] } });
			}

			if (!m.getLayer(d.layer)) {
				m.addLayer({
					id: d.layer,
					type: 'circle',
					source: d.src,
					paint: {
						'circle-radius': ['interpolate', ['linear'], ['zoom'], 10, 4, 15, 8, 18, 12],
						'circle-color': ['case', ['boolean', ['feature-state', 'selected'], false], '#ffffff', d.color],
						'circle-stroke-width': ['case', ['boolean', ['feature-state', 'selected'], false], 3, 2],
						'circle-stroke-color': ['case', ['boolean', ['feature-state', 'selected'], false], d.color, '#ffffff']
					}
				});

				m.on('mouseenter', d.layer, () => {
					m.getCanvas().style.cursor = 'pointer';
				});
				m.on('mouseleave', d.layer, () => {
					m.getCanvas().style.cursor = '';
				});

				m.on('click', d.layer, (e) => {
					e.preventDefault();
					const id = e.features?.[0]?.id;
					const list = d.kind === 'official' ? officialEvents : unofficialEvents;
					const ev = list.find((x) => x.id === id);
					if (ev) selectEvent(ev);
				});
			}
		}

		updateEventVisibility();
	}

	function updateEventVisibility() {
		if (!maps) return;
		['dark', 'light'].forEach((t) => {
			const m = maps[t];
			if (m.getLayer('epins-official')) m.setLayoutProperty('epins-official', 'visibility', showOfficial ? 'visible' : 'none');
			if (m.getLayer('epins-unofficial')) m.setLayoutProperty('epins-unofficial', 'visibility', showUnofficial ? 'visible' : 'none');
		});
	}

	function upsertEventSources() {
		if (!maps) return;
		const officialFeatures = officialEvents
			.map((e) => {
				const coords = getEventCoords(e);
				if (!coords) return null;
				return { type: 'Feature', id: e.id, geometry: { type: 'Point', coordinates: coords }, properties: { kind: 'official' } };
			})
			.filter(Boolean);

		const unofficialFeatures = unofficialEvents
			.map((e) => {
				const coords = getEventCoords(e);
				if (!coords) return null;
				return { type: 'Feature', id: e.id, geometry: { type: 'Point', coordinates: coords }, properties: { kind: 'unofficial' } };
			})
			.filter(Boolean);

		for (const t of ['dark', 'light']) {
			const m = maps[t];
			const o = m.getSource('esrc-official');
			const u = m.getSource('esrc-unofficial');
			if (o) o.setData({ type: 'FeatureCollection', features: officialFeatures });
			if (u) u.setData({ type: 'FeatureCollection', features: unofficialFeatures });
		}
	}

	function clearSelectedEvent() {
		if (!maps || !selectedEvent) {
			selectedEvent = null;
			return;
		}

		for (const t of ['dark', 'light']) {
			const m = maps[t];
			if (m.getSource('esrc-official')) m.setFeatureState({ source: 'esrc-official', id: selectedEvent.id }, { selected: false });
			if (m.getSource('esrc-unofficial')) m.setFeatureState({ source: 'esrc-unofficial', id: selectedEvent.id }, { selected: false });
		}
		selectedEvent = null;
	}

	function selectEvent(ev) {
		clearSelectedEvent();
		selectedEvent = ev;

		const coords = getEventCoords(ev);
		if (!coords) return;

		const src = ev.kind === 'official' ? 'esrc-official' : 'esrc-unofficial';
		for (const t of ['dark', 'light']) {
			if (maps[t].getSource(src)) maps[t].setFeatureState({ source: src, id: ev.id }, { selected: true });
		}

		maps[theme].flyTo({ center: coords, zoom: 17, pitch: 55, bearing: maps[theme].getBearing(), speed: 0.5, curve: 1.4 });
	}

	function flyTo(id) {
		const c = CAMPS[id];
		if (!c) return;
		activeCampus = id;
		selectedBuilding = null;
		clearSelectedEvent();

		if (selectedBldg) {
			['dark', 'light'].forEach((t) => {
				['e', 'b'].forEach((campus) => {
					const src = `bsrc-${campus}`;
					if (maps[t].getSource(src)) {
						maps[t].setFeatureState({ source: src, id: selectedBldg }, { selected: false });
					}
				});
			});
			selectedBldg = null;
		}

		document.querySelectorAll('.blitem').forEach((b) => b.classList.remove('active'));

		document.querySelectorAll('.cbtn').forEach((b) => b.classList.remove('active'));
		document.getElementById(`btn-${id}`)?.classList.add('active');
		document.querySelectorAll('.mrkr').forEach((m) => m.classList.remove('active'));
		document.getElementById(`mk-dark-${id}`)?.classList.add('active');
		document.getElementById(`mk-light-${id}`)?.classList.add('active');

		renderBuildingList(id);

		showBadge(`Flying to ${c.label}…`);
		maps[theme].flyTo({
			center: c.center,
			zoom: c.zoom,
			pitch: c.pitch,
			bearing: c.bearing,
			speed: 0.55,
			curve: 1.5,
			easing: (t) => (t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t)
		});

		const sp = document.getElementById('sp');
		const vp = document.getElementById('vp');
		const sb = document.getElementById('sb');
		const vb = document.getElementById('vb');
		sp.value = c.pitch;
		vp.textContent = `${c.pitch}°`;
		sb.value = c.bearing;
		vb.textContent = `${c.bearing}°`;
		maps[theme].once('moveend', hideBadge);
	}

	function showOverview() {
		document.querySelectorAll('.cbtn').forEach((b) => b.classList.remove('active'));
		document.querySelectorAll('.mrkr').forEach((m) => m.classList.remove('active'));
		selectedBuilding = null;
		clearSelectedEvent();
		['dark', 'light'].forEach((t) => {
			if (campusPopups[t]) {
				campusPopups[t].remove();
				campusPopups[t] = null;
			}
		});

		showBadge('Zooming to Rio Grande Valley…');
		maps[theme].fitBounds(
			[
				[-98.22, 25.852],
				[-97.454, 26.352]
			],
			{
				padding: { top: 80, bottom: 80, left: 300, right: 80 },
				pitch: 30,
				bearing: 0,
				duration: 2200,
				easing: (t) => (t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t)
			}
		);

		const sp = document.getElementById('sp');
		const vp = document.getElementById('vp');
		const sb = document.getElementById('sb');
		const vb = document.getElementById('vb');
		sp.value = 30;
		vp.textContent = '30°';
		sb.value = 0;
		vb.textContent = '0°';
		maps[theme].once('moveend', hideBadge);
	}

	function setPitch(v) {
		maps[theme].easeTo({ pitch: v, duration: 300 });
		document.getElementById('vp').textContent = `${v}°`;
	}

	function setBearing(v) {
		maps[theme].easeTo({ bearing: v, duration: 300 });
		document.getElementById('vb').textContent = `${v}°`;
	}

	function toggleBuildings(on) {
		['dark', 'light'].forEach((t) => {
			if (maps[t].getLayer('bld3d')) maps[t].setLayoutProperty('bld3d', 'visibility', on ? 'visible' : 'none');
		});
	}

	function toggleTerrain(on) {
		terrainOn = on;
		['dark', 'light'].forEach((t) => (on ? maps[t].setTerrain({ source: 'dem', exaggeration: TERRAIN_EXAG }) : maps[t].setTerrain(null)));
	}

	function toggleRotate(on) {
		if (on) {
			rotInt = setInterval(() => {
				maps[theme].easeTo({ bearing: maps[theme].getBearing() + 0.3, duration: 50, easing: (t) => t });
			}, 50);
		} else {
			clearInterval(rotInt);
			rotInt = null;
		}
	}

	function onMapReady(t) {
		const m = maps[t];
		m.addSource('dem', {
			type: 'raster-dem',
			url: 'https://demotiles.maplibre.org/terrain-tiles/tiles.json',
			tileSize: 256,
			encoding: 'mapbox'
		});
		if (terrainOn) m.setTerrain({ source: 'dem', exaggeration: TERRAIN_EXAG });
		addBuildings(m, t);
		Object.entries(CAMPS).forEach(([id, c]) => addCampusMarker(m, t, id, c));
		addBuildingLayers(m, t);
		addEventLayers(m, t);
		ready[t] = true;
		if (ready.dark && ready.light) {
			renderBuildingList('e');
			flyTo('e');
			upsertEventSources();
		}
	}

		onMount(() => {
		document.documentElement.setAttribute('data-theme', theme);

		(async () => {
			const { data, error } = await supabase
				.from('events')
				.select('id, kind, title, description, organization_name, campus, building_code, location_text, start_at, end_at, published_at')
				.order('start_at', { ascending: true, nullsFirst: false });

			if (!error && Array.isArray(data)) {
				officialEvents = data.filter((e) => e.kind === 'official');
				unofficialEvents = data.filter((e) => e.kind === 'unofficial');
				upsertEventSources();
			}
		})();

		maps = {
			dark: new maplibregl.Map({
				container: 'map-dark',
				style: STYLES.dark,
				center: CAMPS.e.center,
				zoom: CAMPS.e.zoom,
				pitch: CAMPS.e.pitch,
				bearing: CAMPS.e.bearing,
				antialias: true
			}),
			light: new maplibregl.Map({
				container: 'map-light',
				style: STYLES.light,
				center: CAMPS.e.center,
				zoom: CAMPS.e.zoom,
				pitch: CAMPS.e.pitch,
				bearing: CAMPS.e.bearing,
				antialias: true
			})
		};

		maps.dark.on('move', () => syncTo(maps.dark, maps.light));
		maps.light.on('move', () => syncTo(maps.light, maps.dark));

		Object.entries(maps).forEach(([t, m]) => {
			m.addControl(new maplibregl.NavigationControl({ visualizePitch: true }), 'bottom-right');
			m.addControl(new maplibregl.ScaleControl({ unit: 'imperial' }), 'bottom-left');
			m.on('load', () => onMapReady(t));
		});

		keyHandler = (e) => {
			if (e.key === '1') flyTo('e');
			if (e.key === '2') flyTo('b');
			if (e.key === '0') showOverview();
			if (e.key === 't' || e.key === 'T') toggleTheme();
			if (e.key === 'r' || e.key === 'R') {
				const cb = document.getElementById('tog-r');
				cb.checked = !cb.checked;
				toggleRotate(cb.checked);
			}
		};
		document.addEventListener('keydown', keyHandler);

			return () => {
				document.removeEventListener('keydown', keyHandler);
				if (rotInt) clearInterval(rotInt);
				Object.values(maps).forEach((m) => m.remove());
			};
		});

	onDestroy(() => {
		if (keyHandler) document.removeEventListener('keydown', keyHandler);
	});
</script>

<svelte:head>
	<title>UTRGV Campus Map — MapLibre</title>
</svelte:head>

<div id="map-dark" class="map-layer active"></div>
<div id="map-light" class="map-layer"></div>

<div id="panel">
	<div class="ph">
		<div class="logo">UTRGV <span>Edinburg &amp; Brownsville · MapLibre</span></div>
		<button id="theme-btn" onclick={toggleTheme} title="Toggle light / dark (T)">
			<span id="theme-icon">{theme === 'dark' ? '☀' : '☾'}</span>
		</button>
	</div>
	<hr />
	<div class="cgrp">
		<div class="slbl">Select campus</div>
		<button class="cbtn active" id="btn-e" onclick={() => flyTo('e')}>
			<span class="cname">Edinburg Campus</span>
			<span class="caddr">1201 W. University Dr., Edinburg TX 78539</span>
		</button>
		<button class="cbtn" id="btn-b" onclick={() => flyTo('b')}>
			<span class="cname">Brownsville Campus</span>
			<span class="caddr">One W. University Blvd., Brownsville TX 78520</span>
		</button>
		<button id="btn-ov" onclick={showOverview}>View both campuses</button>
	</div>
	<hr />
	<div class="sgrp">
		<div class="slbl">Camera</div>
		<div class="srow">
			<div class="slbl2"><span>Pitch (tilt)</span><span class="v" id="vp">60°</span></div>
			<input type="range" id="sp" min="0" max="85" step="1" value="60" oninput={(e) => setPitch(+e.currentTarget.value)} />
		</div>
		<div class="srow">
			<div class="slbl2"><span>Bearing (rotation)</span><span class="v" id="vb">-20°</span></div>
			<input type="range" id="sb" min="-180" max="180" step="1" value="-20" oninput={(e) => setBearing(+e.currentTarget.value)} />
		</div>
	</div>
	<hr />
	<div class="tgrp">
		<div class="slbl">Layers</div>
		<div class="trow">
			<span>3D buildings</span>
			<label class="tog">
				<input type="checkbox" id="tog-b" checked onchange={(e) => toggleBuildings(e.currentTarget.checked)} />
				<span class="tog-track"></span>
				<span class="tog-thumb"></span>
			</label>
		</div>
		<div class="trow">
			<span>Terrain</span>
			<label class="tog">
				<input type="checkbox" id="tog-t" checked onchange={(e) => toggleTerrain(e.currentTarget.checked)} />
				<span class="tog-track"></span>
				<span class="tog-thumb"></span>
			</label>
		</div>
		<div class="trow">
			<span>Auto-rotate</span>
			<label class="tog">
				<input type="checkbox" id="tog-r" onchange={(e) => toggleRotate(e.currentTarget.checked)} />
				<span class="tog-track"></span>
				<span class="tog-thumb"></span>
			</label>
		</div>
	</div>
	<hr />
	<div class="bsect">
		<button class="bsect-hdr" onclick={toggleBuildingList}>
			<span class="slbl">Buildings</span>
			<span class="bsect-arrow" id="barrow">▾</span>
		</button>
		<div id="blist"></div>
	</div>
</div>

<div class="rtog" style:--rtog-right={(selectedBuilding || showOfficial || showUnofficial) ? '288px' : '16px'}>
	<button class="rtbtn" class:active={showOfficial} onclick={toggleOfficial} title="Toggle official events">O</button>
	<button class="rtbtn" class:active={showUnofficial} onclick={toggleUnofficial} title="Toggle unofficial events">U</button>
	<a class="rtbtn" href={`${base}/request`} title="Request an unofficial event">+</a>
</div>

{#if selectedBuilding || showOfficial || showUnofficial}
	<div id="bpanel">
		{#if selectedBuilding}
			<div class="ph">
				<div class="logo">Building <span>{CAMPS[selectedBuilding.campus].label}</span></div>
				<button class="rtbtn" onclick={clearSelectedBuilding} title="Clear selection">×</button>
			</div>
			<hr />
			<div class="cgrp">
				<div class="slbl">{selectedBuilding.name}</div>
				<div class="trow"><span>Code</span><span class="v">{selectedBuilding.code}</span></div>
				<div class="trow"><span>Type</span><span class="v">{selectedBuilding.type}</span></div>
				<div class="trow"><span>Campus</span><span class="v">{selectedBuilding.campus.toUpperCase()}</span></div>
			</div>
		{/if}

		{#if selectedEvent}
			{#if selectedBuilding}
				<hr />
			{/if}
			<div class="ph">
				<div class="logo">Event <span>{selectedEvent.kind}</span></div>
				<button class="rtbtn" onclick={clearSelectedEvent} title="Clear event">×</button>
			</div>
			<hr />
			<div class="cgrp">
				<div class="slbl">{selectedEvent.title}</div>
				{#if selectedEvent.organization_name}
					<div class="empt">{selectedEvent.organization_name}</div>
				{/if}
				{#if selectedEvent.description}
					<div class="empt">{selectedEvent.description}</div>
				{/if}
			</div>
		{/if}

		{#if showOfficial}
			{#if selectedBuilding || selectedEvent}
				<hr />
			{/if}
			<div class="cgrp">
				<div class="slbl">Official events</div>
				{#if officialEvents.length === 0}
					<div class="empt">No official events</div>
				{:else}
					{#each officialEvents as ev (ev.id)}
						<button class="evitem" class:active={selectedEvent?.id === ev.id} onclick={() => selectEvent(ev)}>
							<div class="evtitle">{ev.title}</div>
							<div class="evmeta">{ev.organization_name ?? '—'}</div>
						</button>
					{/each}
				{/if}
			</div>
		{/if}

		{#if showUnofficial}
			{#if selectedBuilding || selectedEvent || showOfficial}
				<hr />
			{/if}
			<div class="cgrp">
				<div class="slbl">Unofficial events</div>
				{#if unofficialEvents.length === 0}
					<div class="empt">No unofficial events</div>
				{:else}
					{#each unofficialEvents as ev (ev.id)}
						<button class="evitem" class:active={selectedEvent?.id === ev.id} onclick={() => selectEvent(ev)}>
							<div class="evtitle">{ev.title}</div>
							<div class="evmeta">{ev.organization_name ?? '—'}</div>
						</button>
					{/each}
				{/if}
			</div>
		{/if}
	</div>
{/if}

<div id="info">
	<strong>UTRGV Campus Map</strong> · MapLibre GL JS · No API key needed<br />
	Click a <strong>◆ diamond pin</strong> or building name to explore · <strong>T</strong> Theme
</div>
<div id="badge"></div>
