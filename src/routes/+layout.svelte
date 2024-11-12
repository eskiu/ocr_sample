<script lang="ts">
	import '../app.css';
	let { children } = $props();
</script>

<main class="flex flex-col min-h-screen bg-slate-900 text-white justify-center items-center">
	{@render children()}
</main>
