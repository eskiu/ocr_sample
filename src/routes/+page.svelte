<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	import { createWorker } from 'tesseract.js';

	let videoElement: HTMLVideoElement;
	let videoStream: MediaStream | null = null;

	// Variables del rectángulo
	let rectX = 100; // Posición inicial en X
	let rectY = 100; // Posición inicial en Y
	let rectWidth = 200; // Ancho inicial
	let rectHeight = 150; // Alto inicial

	// Variables para mover y redimensionar
	let isDragging = false;
	let isResizing = false;
	let dragStartX = 0;
	let dragStartY = 0;
	let resizeStartX = 0;
	let resizeStartY = 0;
	let resizeHandle = ''; // 'tl', 'tr', 'bl', 'br' para las esquinas

	let resultText = '';

	// Aseguramos que `window` esté definido
	onMount(async () => {
		if (typeof window !== 'undefined') {
			try {
				videoStream = await navigator.mediaDevices.getUserMedia({ video: true });
				if (videoElement) {
					videoElement.srcObject = videoStream;
					await videoElement.play();
				}
			} catch (err) {
				console.error('Error al acceder a la cámara:', err);
			}

			window.addEventListener('mousemove', handleMouseMove);
			window.addEventListener('mouseup', handleMouseUp);
			window.addEventListener('touchmove', handleTouchMove, { passive: false });
			window.addEventListener('touchend', handleTouchEnd);
		}
	});

	onDestroy(() => {
		if (typeof window !== 'undefined') {
			window.removeEventListener('mousemove', handleMouseMove);
			window.removeEventListener('mouseup', handleMouseUp);
			window.removeEventListener('touchmove', handleTouchMove);
			window.removeEventListener('touchend', handleTouchEnd);
		}
		if (videoStream) {
			videoStream.getTracks().forEach((track) => track.stop());
		}
	});

	function handleRectMouseDown(event: MouseEvent | TouchEvent) {
		if ((event.target as HTMLElement).classList.contains('resize-handle')) {
			// Si es un handle de redimensionamiento, no hacemos nada aquí
			return;
		}
		isDragging = true;
		const clientX = event instanceof MouseEvent ? event.clientX : event.touches[0].clientX;
		const clientY = event instanceof MouseEvent ? event.clientY : event.touches[0].clientY;
		dragStartX = clientX;
		dragStartY = clientY;
		event.preventDefault();
	}

	function handleResizeMouseDown(event: MouseEvent | TouchEvent, handle: string) {
		isResizing = true;
		resizeHandle = handle;
		const clientX = event instanceof MouseEvent ? event.clientX : event.touches[0].clientX;
		const clientY = event instanceof MouseEvent ? event.clientY : event.touches[0].clientY;
		resizeStartX = clientX;
		resizeStartY = clientY;
		event.stopPropagation(); // Evita que se active el mousedown del rectángulo
		event.preventDefault();
	}

	function handleMouseMove(event: MouseEvent) {
		if (isDragging || isResizing) {
			event.preventDefault();
		}
		if (isDragging) {
			const dx = event.clientX - dragStartX;
			const dy = event.clientY - dragStartY;

			rectX += dx;
			rectY += dy;

			// Asegurarse de que el rectángulo permanezca dentro del video
			const videoRect = videoElement.getBoundingClientRect();
			rectX = Math.max(0, Math.min(rectX, videoRect.width - rectWidth));
			rectY = Math.max(0, Math.min(rectY, videoRect.height - rectHeight));

			dragStartX = event.clientX;
			dragStartY = event.clientY;
		} else if (isResizing) {
			const dx = event.clientX - resizeStartX;
			const dy = event.clientY - resizeStartY;

			resizeRectangle(dx, dy);

			resizeStartX = event.clientX;
			resizeStartY = event.clientY;
		}
	}

	function handleMouseUp(event: MouseEvent) {
		isDragging = false;
		isResizing = false;
	}

	function handleTouchMove(event: TouchEvent) {
		if (isDragging || isResizing) {
			event.preventDefault();
		}
		if (isDragging) {
			const touch = event.touches[0];
			const dx = touch.clientX - dragStartX;
			const dy = touch.clientY - dragStartY;

			rectX += dx;
			rectY += dy;

			// Asegurarse de que el rectángulo permanezca dentro del video
			const videoRect = videoElement.getBoundingClientRect();
			rectX = Math.max(0, Math.min(rectX, videoRect.width - rectWidth));
			rectY = Math.max(0, Math.min(rectY, videoRect.height - rectHeight));

			dragStartX = touch.clientX;
			dragStartY = touch.clientY;
		} else if (isResizing) {
			const touch = event.touches[0];
			const dx = touch.clientX - resizeStartX;
			const dy = touch.clientY - resizeStartY;

			resizeRectangle(dx, dy);

			resizeStartX = touch.clientX;
			resizeStartY = touch.clientY;
		}
	}

	function handleTouchEnd(event: TouchEvent) {
		isDragging = false;
		isResizing = false;
	}

	function resizeRectangle(dx: number, dy: number) {
		if (resizeHandle === 'tl') {
			rectX += dx;
			rectY += dy;
			rectWidth -= dx;
			rectHeight -= dy;
		} else if (resizeHandle === 'tr') {
			rectY += dy;
			rectWidth += dx;
			rectHeight -= dy;
		} else if (resizeHandle === 'bl') {
			rectX += dx;
			rectWidth -= dx;
			rectHeight += dy;
		} else if (resizeHandle === 'br') {
			rectWidth += dx;
			rectHeight += dy;
		}

		// Asegurarse de que el ancho y alto sean positivos y que el rectángulo permanezca dentro del video
		rectWidth = Math.max(20, rectWidth);
		rectHeight = Math.max(20, rectHeight);

		const videoRect = videoElement.getBoundingClientRect();
		rectX = Math.max(0, Math.min(rectX, videoRect.width - rectWidth));
		rectY = Math.max(0, Math.min(rectY, videoRect.height - rectHeight));
	}

	function calculateSelection() {
		// Escalar las coordenadas al tamaño real del video
		const videoWidth = videoElement.videoWidth;
		const videoHeight = videoElement.videoHeight;
		const displayWidth = videoElement.clientWidth;
		const displayHeight = videoElement.clientHeight;

		const scaleX = videoWidth / displayWidth;
		const scaleY = videoHeight / displayHeight;

		const left = rectX * scaleX;
		const top = rectY * scaleY;
		const width = rectWidth * scaleX;
		const height = rectHeight * scaleY;

		return {
			left,
			top,
			width,
			height
		};
	}

	async function captureAndRecognize() {
		if (!videoElement) return;

		const selection = calculateSelection();

		// Crear un canvas para capturar el frame actual
		const tempCanvas = document.createElement('canvas');
		const context = tempCanvas.getContext('2d');
		if (!context) return;

		// Establecer las dimensiones del canvas al tamaño del video
		tempCanvas.width = videoElement.videoWidth;
		tempCanvas.height = videoElement.videoHeight;

		// Dibujar el frame actual en el canvas
		context.drawImage(videoElement, 0, 0, tempCanvas.width, tempCanvas.height);

		// Obtener los datos de la imagen de la selección
		const imageData = context.getImageData(
			selection.left,
			selection.top,
			selection.width,
			selection.height
		);

		// Crear un nuevo canvas con el área seleccionada
		const croppedCanvas = document.createElement('canvas');
		const croppedContext = croppedCanvas.getContext('2d');
		if (!croppedContext) return;

		croppedCanvas.width = selection.width;
		croppedCanvas.height = selection.height;
		croppedContext.putImageData(imageData, 0, 0);

		// Convertir el canvas a Data URL
		const dataURL = croppedCanvas.toDataURL('image/png');

		// Utilizar Tesseract.js para reconocer el texto
		const worker = await createWorker();

		const {
			data: { text }
		} = await worker.recognize(dataURL);
		resultText = text;
		// if (videoStream) videoStream.getTracks().forEach((track) => track.stop());
		await worker.terminate();
	}
</script>

<div class="mx-auto flex w-fit flex-col items-center">
	<div class="video-container">
		<video bind:this={videoElement} width="640" height="480" autoplay playsinline>
			<!-- Añadimos una pista de subtítulos vacía para la accesibilidad -->
			<track kind="captions" srclang="es" label="Español" />
		</video>
		<div
			class="selection-rect"
			style="left: {rectX}px; top: {rectY}px; width: {rectWidth}px; height: {rectHeight}px;"
			role="application"
			aria-label="Área de selección"
			tabindex="0"
			on:mousedown|preventDefault={handleRectMouseDown}
			on:touchstart|preventDefault={handleRectMouseDown}
		>
			<!-- Handles de redimensionamiento -->
			<div
				class="resize-handle top-left"
				role="separator"
				aria-label="Redimensionar esquina superior izquierda"
				tabindex="0"
				on:mousedown|preventDefault={(event) => handleResizeMouseDown(event, 'tl')}
				on:touchstart|preventDefault={(event) => handleResizeMouseDown(event, 'tl')}
			></div>
			<div
				class="resize-handle top-right"
				role="separator"
				aria-label="Redimensionar esquina superior derecha"
				tabindex="0"
				on:mousedown|preventDefault={(event) => handleResizeMouseDown(event, 'tr')}
				on:touchstart|preventDefault={(event) => handleResizeMouseDown(event, 'tr')}
			></div>
			<div
				class="resize-handle bottom-left"
				role="separator"
				aria-label="Redimensionar esquina inferior izquierda"
				tabindex="0"
				on:mousedown|preventDefault={(event) => handleResizeMouseDown(event, 'bl')}
				on:touchstart|preventDefault={(event) => handleResizeMouseDown(event, 'bl')}
			></div>
			<div
				class="resize-handle bottom-right"
				role="separator"
				aria-label="Redimensionar esquina inferior derecha"
				tabindex="0"
				on:mousedown|preventDefault={(event) => handleResizeMouseDown(event, 'br')}
				on:touchstart|preventDefault={(event) => handleResizeMouseDown(event, 'br')}
			></div>
		</div>
	</div>

	<button on:click={captureAndRecognize}>Capturar y Transcribir</button>

	{#if resultText}
		<h3>Texto Reconocido:</h3>
		<pre class="result-text">{resultText}</pre>
	{/if}
</div>

<style>
	.video-container {
		position: relative;
		display: inline-block;
	}

	.selection-rect {
		position: absolute;
		border: 2px solid blue;
		cursor: move;
		box-sizing: border-box;
	}

	.resize-handle {
		width: 20px;
		height: 20px;
		position: absolute;
		background-color: white;
		border: 2px solid black;
	}

	.resize-handle.top-left {
		top: -10px;
		left: -10px;
		cursor: nwse-resize;
	}

	.resize-handle.top-right {
		top: -10px;
		right: -10px;
		cursor: nesw-resize;
	}

	.resize-handle.bottom-left {
		bottom: -10px;
		left: -10px;
		cursor: nesw-resize;
	}

	.resize-handle.bottom-right {
		bottom: -10px;
		right: -10px;
		cursor: nwse-resize;
	}

	.result-text {
		white-space: pre-wrap;
		text-align: left;
	}
</style>
