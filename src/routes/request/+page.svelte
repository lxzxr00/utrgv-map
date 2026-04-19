<script>
	import { goto } from '$app/navigation';
	import { base } from '$app/paths';
	import { supabase } from '$lib/supabaseClient';

	const BUILDINGS = {
		e: [
			{ code: 'ESSBL', name: 'Student Services Building' },
			{ code: 'ELIBR', name: 'University Library' },
			{ code: 'ECOBE', name: 'Robert C. Vackar College of Business' },
			{ code: 'ESCNE', name: 'Science Building' },
			{ code: 'EEDUC', name: 'Education Complex' },
			{ code: 'ECCTR', name: 'Computer Center' },
			{ code: 'EMAGC', name: 'Mathematics & General Classrooms' },
			{ code: 'EPLAN', name: 'H.E.B. Planetarium' },
			{ code: 'EPACC', name: 'Performing Arts Complex C' },
			{ code: 'EPACB', name: 'Performing Arts Complex B' },
			{ code: 'EINNV', name: 'Innovation Building' },
			{ code: 'EACSB', name: 'Academic Services Building' },
			{ code: 'EUREC', name: 'University Recreation Center' },
			{ code: 'EHRTG', name: 'Heritage Hall' },
			{ code: 'ETROX', name: 'Troxel Hall' },
			{ code: 'EIEAB', name: 'Interdisciplinary Engineering & Academic Bldg' },
			{ code: 'EHPE1', name: 'Health & Physical Education Complex' },
			{ code: 'ELCTR', name: 'The Learning Center' },
			{ code: 'ELABS', name: 'Liberal Arts Building South' },
			{ code: 'EEMLH', name: 'Emilia Schunior Ramirez Hall' },
			{ code: 'ESWKH', name: 'Southwick Hall' },
			{ code: 'ECESS', name: 'Community Engagement & Student Success' }
		],
		b: [
			{ code: 'BMAIN', name: 'Main Building (The Tower)' },
			{ code: 'BLIBR', name: 'University Library' },
			{ code: 'BSETB', name: 'Science & Engineering Technology Building' },
			{ code: 'BLHSB', name: 'Life & Health Sciences Building' },
			{ code: 'BLCBR', name: 'Luis V. Colom Biomedical Research Facility' },
			{ code: 'BSTUN', name: 'Student Union Building' }
		]
	};

	let title = $state('');
	let email = $state('');
	let details = $state('');
	let campus = $state('e');
	let buildingCode = $state('');

	let submitting = $state(false);
	let submitted = $state(false);
	let alreadySubmitted = $state(false);
	let touched = $state(false);
	let errorMsg = $state('');

	function storageKey() {
		return `unofficial-request:${email.trim().toLowerCase()}:${campus}:${buildingCode}:${title.trim().toLowerCase()}`;
	}

	function validateEmail(v) {
		return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v.trim());
	}

	function descriptionOk(text) {
		return text.trim().length >= 26;
	}

	function checkAlready() {
		touched = true;
		errorMsg = '';
		if (!title.trim() || !email.trim() || !buildingCode) {
			alreadySubmitted = false;
			return;
		}
		alreadySubmitted = localStorage.getItem(storageKey()) === '1';
	}

	function clearAlready() {
		if (!title.trim() || !email.trim() || !buildingCode) return;
		localStorage.removeItem(storageKey());
		alreadySubmitted = false;
	}

	async function submit() {
		errorMsg = '';
		if (!title.trim()) {
			errorMsg = 'Title is required.';
			return;
		}
		if (!descriptionOk(details)) {
			errorMsg = 'Description must be at least 26 characters.';
			return;
		}
		if (!buildingCode) {
			errorMsg = 'Building is required.';
			return;
		}
		if (!validateEmail(email)) {
			errorMsg = 'Enter a valid email address.';
			return;
		}

		checkAlready();
		if (alreadySubmitted) return;

			submitting = true;
		try {
			const { error } = await supabase.from('events').insert({
				kind: 'unofficial',
				title: title.trim(),
				description: `Submitted by: ${email.trim()}\n\n${details.trim()}`,
				organization_name: null,
				campus,
				building_code: buildingCode,
				location_text: null,
				start_at: null,
				end_at: null
			});
			if (error) {
				errorMsg = error.message;
				return;
			}

			localStorage.setItem(storageKey(), '1');
			submitted = true;
			alreadySubmitted = true;
		} finally {
			submitting = false;
		}
	}
</script>

<svelte:head>
	<title>Request Unofficial Event</title>
</svelte:head>

<div class="reqwrap">
	<div class="reqpanel">
		<div class="ph">
			<div class="logo">Request <span>Unofficial event</span></div>
			<a class="rtbtn" href={base || '/'} title="Back to map">←</a>
		</div>
		<hr />

		<div class="cgrp">
			<div class="slbl">Event title</div>
			<input class="tin" bind:value={title} oninput={checkAlready} placeholder="e.g., Club meeting" />
		</div>

		<div class="cgrp">
			<div class="slbl">Email</div>
			<input class="tin" bind:value={email} oninput={checkAlready} placeholder="name@example.com" />
		</div>

		<div class="cgrp">
			<div class="slbl">Description (min 26 characters)</div>
			<textarea class="tin tarea" bind:value={details} oninput={checkAlready} placeholder="Tell us what this event is about."></textarea>
		</div>

		<div class="cgrp">
			<div class="slbl">Campus</div>
			<select class="tin" bind:value={campus} onchange={() => { buildingCode = ''; checkAlready(); }}>
				<option value="e">Edinburg</option>
				<option value="b">Brownsville</option>
			</select>
		</div>

		<div class="cgrp">
			<div class="slbl">Building</div>
			<select class="tin" bind:value={buildingCode} onchange={checkAlready}>
				<option value="" disabled selected>Select a building…</option>
				{#each BUILDINGS[campus] as b (b.code)}
					<option value={b.code}>{b.code} — {b.name}</option>
				{/each}
			</select>
		</div>

		{#if errorMsg}
			<div class="err">{errorMsg}</div>
		{/if}

		{#if alreadySubmitted && touched}
			<div class="empt">You already sent this request.</div>
			<button id="btn-ov" onclick={clearAlready}>Submit a new request anyway</button>
		{/if}

		<button class="cbtn" disabled={submitting || alreadySubmitted} onclick={submit}>
			<span class="cname">{submitting ? 'Sending…' : submitted ? 'Sent' : 'Send request'}</span>
			<span class="caddr">Adds to Unofficial events automatically</span>
		</button>

		{#if submitted}
			<button id="btn-ov" onclick={() => goto(base || '/')}>Back to map</button>
		{/if}
	</div>
</div>

<style>
	.reqwrap {
		position: absolute;
		inset: 0;
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 24px;
		background: var(--bg-page);
	}

	.reqpanel {
		width: 360px;
		max-width: calc(100vw - 32px);
		background: var(--bg-panel);
		border: 1px solid var(--bdr);
		border-radius: 16px;
		padding: 18px;
		display: flex;
		flex-direction: column;
		gap: 14px;
		box-shadow: 0 8px 36px var(--shadow);
	}

	.tin {
		width: 100%;
		padding: 10px 12px;
		border-radius: 10px;
		border: 1px solid var(--bdr);
		background: var(--bg-btn);
		color: var(--text-1);
		outline: none;
		transition: var(--tr);
	}

	.tarea {
		min-height: 110px;
		resize: vertical;
		line-height: 1.4;
	}

	.tin:focus {
		background: var(--accent-dim);
		border-color: var(--accent-bdr);
	}

	.err {
		color: var(--accent);
		font-size: 12px;
		line-height: 1.4;
		border: 1px solid var(--accent-bdr);
		background: var(--accent-dim);
		padding: 10px 12px;
		border-radius: 10px;
	}
</style>
