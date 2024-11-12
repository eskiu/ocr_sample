<script lang="ts">
	export let resultText: string = '';
</script>

{#if resultText}
	<h3>Texto Extraído:</h3>
	<pre>{resultText}</pre>
{/if}
