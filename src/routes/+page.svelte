<script lang="ts">
	import '../app.css';
	import { onDestroy, onMount } from 'svelte';

	type MessageData = {
		windowId: string;
		posX: number;
		posY: number;
		width: number;
		height: number;
		time: number;
	};

	const id = crypto.randomUUID();
	const otherWindows: Map<string, MessageData> = new Map();
	const tickInterval = 24 / 1000;

	let bc: BroadcastChannel | undefined;
	let interval: ReturnType<typeof setInterval> | undefined;
	let canvasElement: HTMLCanvasElement | null = null;

	function tick() {
		const screenMaxHeight = screen.availHeight;
		const screenMaxWidth = screen.availWidth;
		const data: MessageData = {
			windowId: id,
			posX: window.screenX,
			posY: window.screenY,
			width: window.innerWidth,
			height: window.innerHeight,
			time: Date.now()
		};
		bc?.postMessage(JSON.stringify(data));

		otherWindows.forEach((data, windowId) => {
			// remove data points that are not seen for two ticks
			if ((Date.now() - data.time) / 1000 > 2 * tickInterval) {
				otherWindows.delete(windowId);
			}
		});

		const allDataPoints = [...otherWindows.values(), data];

		if (!canvasElement) return;
		const ctx = canvasElement.getContext('2d');
		if (!ctx) return;

		ctx.canvas.width = data.width;
		ctx.canvas.height = data.height;

		const projectedDataPoints = allDataPoints.map((data) => {
			// project to canvas relative to this window mid point
			return {
				x: data.posX + data.width / 2 - window.screenX,
				y: data.posY + data.height / 2 - window.screenY
			};
		});

		ctx.clearRect(0, 0, canvasElement.width, canvasElement.height);
		ctx.strokeStyle = getComputedStyle(document.documentElement).getPropertyValue('--color');
		ctx.lineWidth = 8;
		projectedDataPoints.forEach((point) => {
			projectedDataPoints.forEach((otherPoint) => {
				if (point.x === otherPoint.x && point.y === otherPoint.y) {
					return;
				}
				ctx.beginPath();
				ctx.moveTo(point.x, point.y);
				ctx.lineTo(otherPoint.x, otherPoint.y);
				ctx.stroke();
			});
		});
	}

	onMount(async () => {
		bc = new BroadcastChannel('channel');

		bc.onmessage = (event: MessageEvent<string>) => {
			const eventData: MessageData = JSON.parse(event.data);
			otherWindows.set(eventData.windowId, eventData);
		};

		interval = setInterval(tick, tickInterval);
	});

	onDestroy(() => {
		bc?.close();
		clearInterval(interval);
	});
</script>

<canvas id="canvas" bind:this={canvasElement}></canvas>
