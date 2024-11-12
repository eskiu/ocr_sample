<script lang="ts">
	export let onImageSelected: (data: { imageFile: File; imageSrc: string }) => void = () => {};

	let imageFile: File | null = null;
	let imageSrc: string | null = null;

	function handleFileChange(event: Event) {
		const target = event.target as HTMLInputElement;
		const file = target.files ? target.files[0] : null;
		if (file) {
			imageFile = file;
			imageSrc = URL.createObjectURL(file);
			onImageSelected({ imageFile, imageSrc });
		}
	}
</script>

<div class="flex flex-col items-center w-fit mx-auto">
	<input type="file" accept="image/*" on:change={handleFileChange} />
	{#if imageSrc}
		<img src={imageSrc} alt="Imagen seleccionada" width="700" />
	{/if}
</div>
