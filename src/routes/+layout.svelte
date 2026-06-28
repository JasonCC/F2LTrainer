<script lang="ts">
	import '../app.css';
	import { browser } from '$app/environment';

	// pwa prompt state centralized in $lib/pwaPrompt.svelte - used by components
	import { startPwaManager } from '$lib/pwa/pwaManager.svelte';
	import { startWakeLockManager } from '$lib/wakeLock/wakeLockManager.svelte';
	import { casesState, CASES_STATE_STORAGE_KEY } from '$lib/casesState.svelte';
	import { globalState, GLOBAL_STATE_STORAGE_KEY } from '$lib/globalState.svelte';

	import { saveToLocalStorage } from '$lib/utils/localStorage';
	import ToastContainer from '$lib/components/ToastContainer.svelte';

	// Import the whole $env/static/public namespace instead of named exports.
	// A named import of an unset key (e.g. no .env) fails at module LINK time,
	// before any guard can run, which silently hangs the app on the initial
	// loader. A namespace import links fine and yields `undefined` for unset
	// keys, while Vite still inlines real values at build time (so production
	// cloud sync keeps working).
	import * as publicEnv from '$env/static/public';
	import { setupConvex } from 'convex-svelte';
	import { ClerkProvider } from 'svelte-clerk';
	import ConvexClerkSync from './ConvexClerkSync.svelte';

	let { children } = $props();

	// Cast to a loose record: the generated $env/static/public types only include
	// keys that exist at sync time, so direct access wouldn't type-check when a
	// var is unset (e.g. no .env). The values are still inlined at build time.
	const env = publicEnv as Record<string, string | undefined>;
	const PUBLIC_CONVEX_URL = env.PUBLIC_CONVEX_URL;
	const PUBLIC_CLERK_PUBLISHABLE_KEY = env.PUBLIC_CLERK_PUBLISHABLE_KEY;

	// Convex requires a deployment URL. If PUBLIC_CONVEX_URL is missing or
	// malformed (e.g. no .env file), setupConvex() throws at module load — which
	// stops the whole app from mounting and leaves the initial HTML loader on
	// screen forever. Guard it so a misconfigured environment logs a clear error
	// and the app still boots (with cloud sync disabled) instead of hanging.
	if (typeof PUBLIC_CONVEX_URL === 'string' && /^https?:\/\//.test(PUBLIC_CONVEX_URL)) {
		setupConvex(PUBLIC_CONVEX_URL);
	} else {
		console.error(
			'[F2LTrainer] PUBLIC_CONVEX_URL is missing or invalid — cloud sync is disabled. ' +
				'Copy .env.example to .env and set PUBLIC_CONVEX_URL to your Convex deployment ' +
				'URL, then restart the dev server.'
		);
		// Disabled placeholder client so useConvexClient() still resolves in child
		// components and the app can mount instead of crashing.
		setupConvex('https://placeholder.convex.cloud', { disabled: true });
	}

	if (browser && !PUBLIC_CLERK_PUBLISHABLE_KEY) {
		console.warn(
			'[F2LTrainer] PUBLIC_CLERK_PUBLISHABLE_KEY is missing — sign-in and cloud sync ' +
				'are disabled. See .env.example to enable them.'
		);
	}

	if (browser) {
		$effect(() => {
			saveToLocalStorage(GLOBAL_STATE_STORAGE_KEY, globalState);
		});

		// Only persist case states to localStorage when not syncing to avoid conflicts
		$effect(() => {
			if (!globalState.isSyncing) {
				saveToLocalStorage(CASES_STATE_STORAGE_KEY, casesState);
			}
		});

		// Initialize the PWA manager which centralizes service worker
		// registration and `beforeinstallprompt` handling.
		startPwaManager();

		// Initialize the wake lock manager to prevent screen from sleeping
		$effect(() => {
			const cleanup = startWakeLockManager();
			return cleanup;
		});

		// Remove the initial HTML loader once the app is mounted
		$effect(() => {
			if (typeof window !== 'undefined' && (window as any).completeLoadingProgress) {
				(window as any).completeLoadingProgress();
			}

			const loader = document.getElementById('initial-loader');
			if (loader) {
				// Wait for the progress bar to visually hit 100% before fading out
				setTimeout(() => {
					loader.style.opacity = '0';
					setTimeout(() => loader.remove(), 400); // Matches the CSS transition time
				}, 250);
			}
		});
	}
</script>

<ClerkProvider
	publishableKey={PUBLIC_CLERK_PUBLISHABLE_KEY}
	appearance={{
		cssLayerName: 'clerk'
	}}
	localization={{
		signIn: {
			start: {
				title: 'Welcome to F2L Trainer',
				subtitle:
					'Sign in to sync your progress across devices, track your solve history, and never lose your training data.'
			}
		},
		signUp: {
			start: {
				title: 'Join F2L Trainer',
				subtitle:
					'Create an account to unlock cloud sync, cross-device progress tracking, and detailed solve statistics.'
			}
		}
	}}
>
	<ConvexClerkSync />
	{@render children()}
</ClerkProvider>

<ToastContainer />
