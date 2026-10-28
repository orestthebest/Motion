<script>
	import { onDestroy } from 'svelte';

	// --- Settings ---
	const SHAKE_THRESHOLD = 15; // m/s² – how hard you have to shake
	const STILL_THRESHOLD = 2; // m/s² – below this the phone counts as "lying still"
	const STILL_TIME = 600; // ms the phone must be still before the result is fixed
	let tolerance = $state(35); // ° – how far you can tilt and still get the same face

	// --- State ---
	let started = $state(false);
	let rolling = $state(false);
	let value = $state(null); // the fixed dice result
	let shown = $state(null); // what's on screen (random while rolling)
	let hint = $state('Shake your phone to roll');
	let beta = 0;
	let gamma = 0;
	let stillSince = 0;
	let animTimer;

	// Pip positions (3x3 grid, index 0–8) for each face
	const PIPS = {
		1: [4],
		2: [0, 8],
		3: [0, 4, 8],
		4: [0, 2, 6, 8],
		5: [0, 2, 4, 6, 8],
		6: [0, 2, 3, 5, 6, 8]
	};

	// Turns beta/gamma into a dice face, or null if tilted beyond the tolerance.
	// "up" = which side of the phone points to the sky (unit vector in device coords).
	function faceFromOrientation(b, g) {
		const rad = Math.PI / 180;
		const up = {
			x: -Math.sin(g * rad) * Math.cos(b * rad),
			y: Math.sin(b * rad),
			z: Math.cos(b * rad) * Math.cos(g * rad)
		};
		// opposite faces add up to 7, like a real die
		const faces = [
			{ n: 6, v: up.z }, // screen up
			{ n: 1, v: -up.z }, // screen down
			{ n: 2, v: up.y }, // standing upright (portrait)
			{ n: 5, v: -up.y }, // standing upside down
			{ n: 3, v: -up.x }, // on its left side
			{ n: 4, v: up.x } // on its right side
		];
		const best = faces.reduce((a, c) => (c.v > a.v ? c : a));
		// v = cos(angle to the face axis) → inside tolerance if angle <= tolerance
		return best.v >= Math.cos(tolerance * rad) ? best.n : null;
	}

	function onOrientation(e) {
		beta = e.beta ?? 0;
		gamma = e.gamma ?? 0;
	}

	function onMotion(e) {
		// acceleration without gravity; fallback: with gravity minus 9.81
		const a = e.acceleration;
		let force;
		if (a && a.x !== null) {
			force = Math.hypot(a.x, a.y, a.z);
		} else {
			const g = e.accelerationIncludingGravity;
			force = Math.abs(Math.hypot(g.x, g.y, g.z) - 9.81);
		}

		if (force > SHAKE_THRESHOLD) {
			startRoll();
			stillSince = 0;
			return;
		}

		if (!rolling) return;

		// wait until the phone lies still, then read the orientation
		const now = performance.now();
		if (force < STILL_THRESHOLD) {
			if (!stillSince) stillSince = now;
			if (now - stillSince > STILL_TIME) {
				const face = faceFromOrientation(beta, gamma);
				if (face) finishRoll(face);
				else hint = 'Hold the phone on one side';
			}
		} else {
			stillSince = 0;
		}
	}

	function startRoll() {
		if (rolling) return;
		rolling = true;
		value = null;
		hint = 'Rolling…';
		animTimer = setInterval(() => (shown = Math.floor(Math.random() * 6) + 1), 90);
	}

	function finishRoll(face) {
		clearInterval(animTimer);
		rolling = false;
		value = face;
		shown = face;
		hint = 'Shake again for a new number';
		navigator.vibrate?.(60);
	}

	async function start() {
		// iOS needs permission, requested from a tap
		try {
			if (typeof DeviceMotionEvent?.requestPermission === 'function') {
				if ((await DeviceMotionEvent.requestPermission()) !== 'granted') {
					hint = 'Motion permission denied';
					return;
				}
				await DeviceOrientationEvent.requestPermission?.();
			}
		} catch {
			hint = 'Needs HTTPS and a phone with sensors';
			return;
		}
		window.addEventListener('deviceorientation', onOrientation);
		window.addEventListener('devicemotion', onMotion);
		started = true;
	}

	onDestroy(() => {
		if (typeof window === 'undefined') return;
		window.removeEventListener('deviceorientation', onOrientation);
		window.removeEventListener('devicemotion', onMotion);
		clearInterval(animTimer);
	});
</script>

<main class="flex min-h-dvh flex-col items-center justify-center gap-10 bg-zinc-900 p-6 text-white">
	<h1 class="text-2xl font-bold tracking-wide">Motion Dice</h1>

	<div
		class="grid size-48 grid-cols-3 grid-rows-3 gap-2 rounded-3xl bg-white p-6 shadow-2xl transition-transform"
		class:animate-bounce={rolling}
	>
		{#each [0, 1, 2, 3, 4, 5, 6, 7, 8] as i (i)}
			<div class="flex items-center justify-center">
				{#if shown && PIPS[shown].includes(i)}
					<span class="size-9 rounded-full {shown === 1 ? 'bg-red-600' : 'bg-zinc-900'}"></span>
				{/if}
			</div>
		{/each}
	</div>

	<p class="text-6xl font-bold tabular-nums">{value ?? (rolling ? '…' : '–')}</p>

	{#if started}
		<p class="text-zinc-400">{hint}</p>
		<label class="flex w-64 flex-col gap-1 text-sm text-zinc-400">
			Tolerance: {tolerance}°
			<input type="range" min="10" max="45" bind:value={tolerance} />
		</label>
	{:else}
		<button
			class="rounded-xl bg-white px-6 py-3 font-semibold text-zinc-900 active:scale-95"
			onclick={start}
		>
			Start
		</button>
		<p class="text-sm text-zinc-500">{hint}</p>
	{/if}
</main>