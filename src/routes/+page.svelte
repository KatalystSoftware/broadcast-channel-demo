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
	let requestID: ReturnType<typeof requestAnimationFrame> | undefined;
	let canvasElement: HTMLCanvasElement | null = null;

	const tick: FrameRequestCallback = (time) => {
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
		bc?.postMessage(data);

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
				id: data.windowId,
				x: data.posX + data.width / 2 - window.screenX,
				y: data.posY + data.height / 2 - window.screenY
			};
		});

		ctx.clearRect(0, 0, canvasElement.width, canvasElement.height);
		// ctx.strokeStyle = getComputedStyle(document.documentElement).getPropertyValue('--color');
		ctx.lineWidth = 8;
		projectedDataPoints.forEach((point) => {
			projectedDataPoints.forEach((otherPoint) => {
				if (point.id === otherPoint.id) {
					return;
				}
				const color = point.id.slice(0, 6);
				const otherColor = otherPoint.id.slice(0, 6);
				const gradient = ctx.createLinearGradient(point.x, point.y, otherPoint.x, otherPoint.y);
				gradient.addColorStop(0, `#${color}`);
				gradient.addColorStop(1, `#${otherColor}`);
				ctx.strokeStyle = gradient;

				ctx.beginPath();
				ctx.moveTo(point.x, point.y);
				ctx.lineTo(otherPoint.x, otherPoint.y);
				ctx.stroke();
			});
		});

		requestID = requestAnimationFrame(tick);
	};

	onMount(async () => {
		bc = new BroadcastChannel('channel');

		bc.onmessage = (event: MessageEvent<MessageData>) => {
			otherWindows.set(event.data.windowId, event.data);
		};

		requestID = requestAnimationFrame(tick);
	});

	onDestroy(() => {
		bc?.close();
		requestID && cancelAnimationFrame(requestID);
	});
</script>

<canvas id="canvas" bind:this={canvasElement}></canvas>
