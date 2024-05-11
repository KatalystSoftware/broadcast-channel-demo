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
			// Remove window if it hasn't sent a message in 2 seconds
			if (Date.now() - data.time > 2000) {
				otherWindows.delete(windowId);
			}
		});

		const midPointsScreen = Array.from(otherWindows.values()).map((data) => {
			return {
				x: data.posX + data.width / 2,
				y: data.posY + data.height / 2
			};
		});
		const myMidPointScreen = {
			x: window.screenX + window.innerWidth / 2,
			y: window.screenY + window.innerHeight / 2
		};

		if (!canvasElement) return;
		const ctx = canvasElement.getContext('2d');
		if (!ctx) return;

		ctx.canvas.width = data.width;
		ctx.canvas.height = data.height;

		const midPoints = midPointsScreen.map((midPoint) => {
			return {
				x: (midPoint.x - myMidPointScreen.x) * (data.width / screenMaxWidth) + data.width / 2,
				y: (midPoint.y - myMidPointScreen.y) * (data.height / screenMaxHeight) + data.height / 2
			};
		});
		const myMidPoint = {
			x: data.width / 2,
			y: data.height / 2
		};

		ctx.clearRect(0, 0, canvasElement.width, canvasElement.height);
		ctx.strokeStyle = 'black';
		ctx.lineWidth = 8;
		midPoints.forEach((midPoint) => {
			ctx.beginPath();
			ctx.moveTo(myMidPoint.x, myMidPoint.y);
			ctx.lineTo(midPoint.x, midPoint.y);
			ctx.stroke();
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
