<script lang="ts">
	import favicon from '$lib/assets/favicon.svg';

	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
	<link rel="preconnect" href="https://fonts.googleapis.com" />
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
	<link href="https://fonts.googleapis.com/css2?family=Noto+Sans+KR:wght@400;700&display=swap" rel="stylesheet" />
	<style>
		body {
			--background: #f1f1f5;
			--color: #111;
			--border-color: #bbb;
			--font: 'Noto Sans KR', sans-serif;

			background: var(--background);
			color: var(--color);
			font-family: var(--font);
			margin: 0;
		}

		input {
			background: transparent;
			border: 1px solid var(--border-color);
			border-radius: 4px;
			padding: 1rem;
			color: var(--color);
			font-size: 1rem;
			font-family: var(--font);
			background-color: white;
		}
	</style>
</svelte:head>

{@render children()}
