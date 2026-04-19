<script>
	import { base } from '$app/paths';
	import { goto } from '$app/navigation';
	import { onMount } from 'svelte';
	import { supabase } from '$lib/supabaseClient';

	let email = $state('');
	let password = $state('');
	let errorMsg = $state('');
	let loading = $state(true);
	let signingIn = $state(false);
	let isAdmin = $state(false);
	let tickets = $state([]);
	let promotingId = $state(null);

	async function loadAdminState() {
		loading = true;
		errorMsg = '';
		isAdmin = false;
		tickets = [];

		const { data: userData } = await supabase.auth.getUser();
		const user = userData?.user;
		if (!user) {
			loading = false;
			return;
		}

		const { data: profile, error: profileError } = await supabase
			.from('profiles')
			.select('role')
			.eq('id', user.id)
			.maybeSingle();

		if (profileError) {
			errorMsg = profileError.message;
			loading = false;
			return;
		}

		if (profile?.role !== 'admin') {
			errorMsg = 'Not authorized.';
			loading = false;
			return;
		}

		isAdmin = true;
		const { data, error } = await supabase
			.from('events')
			.select('id, kind, title, description, organization_name, campus, building_code, published_at')
			.eq('kind', 'unofficial')
			.order('published_at', { ascending: false, nullsFirst: false });

		if (error) {
			errorMsg = error.message;
			loading = false;
			return;
		}

		tickets = data ?? [];
		loading = false;
	}

	async function signIn() {
		errorMsg = '';
		signingIn = true;
		try {
			const { error } = await supabase.auth.signInWithPassword({
				email: email.trim(),
				password
			});
			if (error) {
				errorMsg = error.message;
				return;
			}
			await loadAdminState();
		} finally {
			signingIn = false;
		}
	}

	async function signOut() {
		await supabase.auth.signOut();
		await loadAdminState();
	}

	async function promote(ticket) {
		promotingId = ticket.id;
		errorMsg = '';
		try {
			const { error } = await supabase.from('events').update({ kind: 'official' }).eq('id', ticket.id);
			if (error) {
				errorMsg = error.message;
				return;
			}
			tickets = tickets.filter((t) => t.id !== ticket.id);
		} finally {
			promotingId = null;
		}
	}

	onMount(() => {
		loadAdminState();
	});
</script>

<svelte:head>
	<title>Admin</title>
</svelte:head>

<div class="reqwrap">
	<div class="reqpanel">
		<div class="ph">
			<div class="logo">Admin <span>Unofficial tickets</span></div>
			<a class="rtbtn" href={base || '/'} title="Back to map">←</a>
		</div>
		<hr />

		{#if errorMsg}
			<div class="err">{errorMsg}</div>
		{/if}

		{#if loading}
			<div class="empt">Loading…</div>
		{:else if !isAdmin}
			<div class="cgrp">
				<div class="slbl">Email</div>
				<input class="tin" bind:value={email} placeholder="admin@example.com" />
			</div>

			<div class="cgrp">
				<div class="slbl">Password</div>
				<input class="tin" type="password" bind:value={password} />
			</div>

			<button class="cbtn" disabled={signingIn} onclick={signIn}>
				<span class="cname">{signingIn ? 'Signing in…' : 'Sign in'}</span>
				<span class="caddr">Admins only</span>
			</button>
		{:else}
			<button id="btn-ov" onclick={signOut}>Sign out</button>
			<hr />

			{#if tickets.length === 0}
				<div class="empt">No unofficial tickets.</div>
			{:else}
				{#each tickets as t (t.id)}
					<div class="evitem" style="cursor: default">
						<div class="evtitle">{t.title}</div>
						<div class="evmeta">{t.campus?.toUpperCase() ?? ''}{t.building_code ? ` · ${t.building_code}` : ''}</div>
						<button id="btn-ov" disabled={promotingId === t.id} onclick={() => promote(t)}>
							{promotingId === t.id ? 'Promoting…' : 'Promote to official'}
						</button>
					</div>
				{/each}
			{/if}

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
		width: 420px;
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

	.tin::placeholder {
		color: var(--text-3);
	}
</style>
