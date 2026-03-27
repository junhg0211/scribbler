<script lang="ts">
	import favicon from '$lib/assets/favicon.svg';

	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
	<style>
		@font-face {
			font-family: 'A2z';
			src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/2601-6@1.0/에이투지체-1Thin.woff2')
				format('woff2');
			font-weight: 100;
			font-display: swap;
		}

		@font-face {
			font-family: 'A2z';
			src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/2601-6@1.0/에이투지체-2ExtraLight.woff2')
				format('woff2');
			font-weight: 200;
			font-display: swap;
		}

		@font-face {
			font-family: 'A2z';
			src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/2601-6@1.0/에이투지체-3Light.woff2')
				format('woff2');
			font-weight: 300;
			font-display: swap;
		}

		@font-face {
			font-family: 'A2z';
			src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/2601-6@1.0/에이투지체-4Regular.woff2')
				format('woff2');
			font-weight: 400;
			font-display: swap;
		}

		@font-face {
			font-family: 'A2z';
			src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/2601-6@1.0/에이투지체-5Medium.woff2')
				format('woff2');
			font-weight: 500;
			font-display: swap;
		}

		@font-face {
			font-family: 'A2z';
			src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/2601-6@1.0/에이투지체-6SemiBold.woff2')
				format('woff2');
			font-weight: 600;
			font-display: swap;
		}

		@font-face {
			font-family: 'A2z';
			src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/2601-6@1.0/에이투지체-7Bold.woff2')
				format('woff2');
			font-weight: 700;
			font-display: swap;
		}

		@font-face {
			font-family: 'A2z';
			src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/2601-6@1.0/에이투지체-8ExtraBold.woff2')
				format('woff2');
			font-weight: 800;
			font-display: swap;
		}

		@font-face {
			font-family: 'A2z';
			src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/2601-6@1.0/에이투지체-9Black.woff2')
				format('woff2');
			font-weight: 900;
			font-display: swap;
		}

		body {
			--background: #f1f1f5;
			--color: #111;
			--border-color: #bbb;
			--font: 'A2z', sans-serif;

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
