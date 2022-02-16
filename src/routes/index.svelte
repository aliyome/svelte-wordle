<script lang="ts">
	import '../app.css';
	import Wordle from '$lib/components/Wordle.svelte';
</script>

<Wordle />

<style>
	:global(html, body) {
		margin: 0;
		height: 100%;
		overflow: hidden;
	}
</style>
