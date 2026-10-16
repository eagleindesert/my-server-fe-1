<script lang="ts">
	import './layout.css';
	import favicon from '$lib/assets/cute-squirrel-with-acorn.png';

	let { children } = $props();
</script>

<svelte:head>
	<title>Tory's Server</title>
	<link rel="icon" type="image/png" href={favicon} />
</svelte:head>
{@render children()}
