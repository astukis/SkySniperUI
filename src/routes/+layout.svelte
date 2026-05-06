<script>
	import favicon from '$lib/assets/favicon.ico';
	import '../app.css';
	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
	<title>Sky Sniper</title>
</svelte:head>

{@render children()}
