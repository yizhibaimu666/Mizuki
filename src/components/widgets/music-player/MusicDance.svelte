<script lang="ts">
	import { onMount } from "svelte";

	// ==================== 可编辑配置区 ====================
	const CONFIG = {
		musicUrl: "/music/dance.mp3", // 音乐文件路径
		// 悬浮视频配置
		videoSrc: "/video/dance.mp4", // 视频路径
		videoWidth: 720, // 视频宽度（px）
		videoHeight: 405, // 视频高度（px）
		videoSpeed: 500, // 移动速度（px/s）
		videoOpacity: 0.8, // 透明度（0～1）
		videoEdgePadding: 8, // 距离窗口边缘的间距（px）
		videoZIndex: 9999, // 悬浮层级
		videoTurnAngle: 55, // 碰边时随机偏转角度（度）

		flipDuration: 500, // 左右翻转运动时间，毫秒
		flipGap: 500, // 翻转停顿时间，毫秒

		stretchDuration: 500, // 上下拉伸运动时间，毫秒
		stretchGap: 500, // 拉伸停顿时间，毫秒

		stretchMax: 1.25, // 最大拉伸倍率

		// 动画目标：尽量选择页面组件，而非每个文字节点
		selector: [
			// 页面主要区域
			"header",
			"main",
			"footer",
			"nav",
			"section",
			"article",
			"aside",

			// 左侧栏和右侧栏的外层及内部组件
			"#sidebar",
			"#left-sidebar-sticky",
			"#sidebar .sidebar-content > *",
			"#left-sidebar-sticky .sidebar-content > *",
			"#right-sidebar",
			"#right-sidebar-sticky",
			"#right-sidebar .sidebar-content > *",
			"#right-sidebar-sticky .sidebar-content > *",

			// 其他卡片
			".card",
			".widget",
			".post-card",
			".card-grid",
		].join(","),
	};
	// =====================================================

	let audio: HTMLAudioElement;
	let playing = $state(false);
	let errorMessage = $state("");
	let video: HTMLVideoElement;
	let videoX = 0;
	let videoY = 0;
	let velocityX = 0;
	let velocityY = 0;
	let videoFrame = 0;
	let lastFrameTime = 0;

	let animations: Animation[] = [];
	let generation = 0;

	// 记录动画前的内联样式，结束后恢复
	const originalStyles = new Map<
		HTMLElement,
		{ translate: string; rotate: string; scale: string }
	>();

	function getTargets(): HTMLElement[] {
		return Array.from(
			document.querySelectorAll<HTMLElement>(CONFIG.selector),
		).filter((element) => {
			if (element.closest("#mzk-dance-control")) return false;

			const style = getComputedStyle(element);

			return (
				style.display !== "none" &&
				style.visibility !== "hidden" &&
				element.getClientRects().length > 0
			);
		});
	}

	function saveOriginalStyle(element: HTMLElement) {
		if (originalStyles.has(element)) return;

		originalStyles.set(element, {
			translate: element.style.translate,
			rotate: element.style.rotate,
			scale: element.style.scale,
		});
	}

	function animate(
		element: HTMLElement,
		property: "rotate" | "scale" | "translate",
		values: string[],
		duration: number,
		gap: number,
	): Animation {
		const active = Math.max(1, duration);
		const pause = Math.max(0, gap);
		const cycle = active + pause;
		const activeRatio = active / cycle;

		const keyframes: Keyframe[] = values.map((value, index) => ({
			[property]: value,
			offset: (activeRatio * index) / (values.length - 1),
		}));

		keyframes.push({
			[property]: values[0],
			offset: 1,
		});

		return element.animate(keyframes, {
			duration: cycle,
			iterations: Infinity,
			easing: "ease-in-out",
			fill: "both",
		});
	}

	function restore() {
		for (const animation of animations) {
			animation.cancel();
		}
		animations = [];

		for (const [element, style] of originalStyles) {
			element.style.translate = style.translate;
			element.style.rotate = style.rotate;
			element.style.scale = style.scale;
		}

		originalStyles.clear();
	}
	// 将视频坐标限制在视口内，并应用到 fixed 视频元素。
	function setVideoPosition() {
		if (!video) return;
		video.style.left = `${videoX}px`;
		video.style.top = `${videoY}px`;
	}

	function randomizeVideoVelocity() {
		const angle = Math.random() * Math.PI * 2;
		const speed = Math.max(1, CONFIG.videoSpeed);
		velocityX = Math.cos(angle) * speed;
		velocityY = Math.sin(angle) * speed;
	}

	// 每帧移动视频；触碰窗口边缘时反弹，并略微随机改变方向。
	function moveFloatingVideo(timestamp: number) {
		if (!playing || !video) return;

		if (lastFrameTime === 0) lastFrameTime = timestamp;
		const delta = Math.min((timestamp - lastFrameTime) / 1000, 0.05);
		lastFrameTime = timestamp;

		const padding = Math.max(0, CONFIG.videoEdgePadding);
		const videoWidth =
			video.getBoundingClientRect().width || CONFIG.videoWidth;
		const videoHeight =
			video.getBoundingClientRect().height || CONFIG.videoHeight;
		const maxX = Math.max(
			padding,
			window.innerWidth - videoWidth - padding,
		);
		const maxY = Math.max(
			padding,
			window.innerHeight - videoHeight - padding,
		);
		videoX += velocityX * delta;
		videoY += velocityY * delta;

		let hitX = false;
		let hitY = false;
		if (videoX <= padding) {
			videoX = padding;
			velocityX = Math.abs(velocityX);
			hitX = true;
		} else if (videoX >= maxX) {
			videoX = maxX;
			velocityX = -Math.abs(velocityX);
			hitX = true;
		}
		if (videoY <= padding) {
			videoY = padding;
			velocityY = Math.abs(velocityY);
			hitY = true;
		} else if (videoY >= maxY) {
			videoY = maxY;
			velocityY = -Math.abs(velocityY);
			hitY = true;
		}

		if (hitX || hitY) {
			const baseAngle = Math.atan2(velocityY, velocityX);
			const maxTurn =
				(Math.max(0, CONFIG.videoTurnAngle) * Math.PI) / 180;
			const angle = baseAngle + (Math.random() * 2 - 1) * maxTurn;
			const speed = Math.max(1, CONFIG.videoSpeed);
			velocityX = Math.cos(angle) * speed;
			velocityY = Math.sin(angle) * speed;
		}

		setVideoPosition();
		videoFrame = requestAnimationFrame(moveFloatingVideo);
	}

	function startFloatingVideo() {
		if (!video) return;
		cancelAnimationFrame(videoFrame);
		const padding = Math.max(0, CONFIG.videoEdgePadding);
		const videoWidth =
			video.getBoundingClientRect().width || CONFIG.videoWidth;
		const videoHeight =
			video.getBoundingClientRect().height || CONFIG.videoHeight;
		const maxX = Math.max(
			padding,
			window.innerWidth - videoWidth - padding,
		);
		const maxY = Math.max(
			padding,
			window.innerHeight - videoHeight - padding,
		);
		videoX = padding + Math.random() * Math.max(0, maxX - padding);
		videoY = padding + Math.random() * Math.max(0, maxY - padding);
		randomizeVideoVelocity();
		lastFrameTime = 0;
		setVideoPosition();
		videoFrame = requestAnimationFrame(moveFloatingVideo);
	}

	function stopFloatingVideo() {
		cancelAnimationFrame(videoFrame);
		videoFrame = 0;
		lastFrameTime = 0;
		if (video) video.pause();
	}

	function resizeFloatingVideo() {
		if (!video) return;

		const padding = Math.max(0, CONFIG.videoEdgePadding);
		// 窄窗口下按原始宽高比缩小视频，确保整个视频始终留在视口内。
		const scale = Math.min(
			1,
			Math.max(
				0.1,
				(window.innerWidth - padding * 2) / CONFIG.videoWidth,
			),
			Math.max(
				0.1,
				(window.innerHeight - padding * 2) / CONFIG.videoHeight,
			),
		);
		video.style.width = `${CONFIG.videoWidth * scale}px`;
		video.style.height = `${CONFIG.videoHeight * scale}px`;
		const videoWidth =
			video.getBoundingClientRect().width || CONFIG.videoWidth;
		const videoHeight =
			video.getBoundingClientRect().height || CONFIG.videoHeight;
		const maxX = Math.max(
			padding,
			window.innerWidth - videoWidth - padding,
		);
		const maxY = Math.max(
			padding,
			window.innerHeight - videoHeight - padding,
		);

		videoX = Math.max(padding, Math.min(videoX, maxX));
		videoY = Math.max(padding, Math.min(videoY, maxY));

		setVideoPosition();
	}
	function stopDance(resetAudio = true) {
		generation++;
		playing = false;
		stopFloatingVideo();

		restore();

		if (audio) {
			audio.pause();

			if (resetAudio) {
				try {
					audio.currentTime = 0;
				} catch {
					// 音频尚未就绪时忽略
				}
			}
		}
	}

	async function startDance() {
		if (playing) return;

		const currentGeneration = ++generation;
		errorMessage = "";

		try {
			await audio.play();

			if (currentGeneration !== generation) return;

			playing = true;
			// 启动悬浮视频
			startFloatingVideo();

			video.play().catch((error) => {
				console.warn("[MusicDance] 视频播放失败：", error);
			});

			const targets = getTargets();
			console.log("[MusicDance] 找到动画目标数量：", targets.length);
			console.log("[MusicDance] 动画目标：", targets);

			for (const element of targets) {
				saveOriginalStyle(element);

				// 以组件中心为轴进行 3D 左右翻转
				const flip = animate(
					element,
					"rotate",
					["y 0deg", "y 180deg", "y 360deg"],
					CONFIG.flipDuration,
					CONFIG.flipGap,
				);

				// 从中心缩放，同时向下补偿位移，让底部尽量保持不动
				const maxScale = Math.max(1, CONFIG.stretchMax);
				const extraHeight =
					(element.getBoundingClientRect().height * (maxScale - 1)) /
					2;

				const stretch = animate(
					element,
					"scale",
					["1 1", `1 ${maxScale}`, "1 1"],
					CONFIG.stretchDuration,
					CONFIG.stretchGap,
				);

				const anchor = animate(
					element,
					"translate",
					["0px 0px", `0px ${extraHeight}px`, "0px 0px"],
					CONFIG.stretchDuration,
					CONFIG.stretchGap,
				);

				animations.push(flip, stretch, anchor);
			}
		} catch (error) {
			console.error("[MusicDance] 播放失败：", error);
			stopDance();
			errorMessage = "音乐加载失败，请检查音乐路径";
		}
	}

	onMount(() => {
		audio.src = CONFIG.musicUrl;

		// 将视频提升到 body 直属子元素，避免祖先元素的 transform
		// 把 position: fixed 的定位范围从浏览器视口改成页面区域。
		if (video && video.parentElement !== document.body) {
			document.body.appendChild(video);
		}
		video.src = CONFIG.videoSrc;
		resizeFloatingVideo();

		window.addEventListener("resize", resizeFloatingVideo);

		const handleEnded = () => stopDance();
		const handlePageHide = () => stopDance();

		audio.addEventListener("ended", handleEnded);
		window.addEventListener("pagehide", handlePageHide);

		return () => {
			stopDance();
			window.removeEventListener("resize", resizeFloatingVideo);
			cancelAnimationFrame(videoFrame);
			audio.removeEventListener("ended", handleEnded);
			window.removeEventListener("pagehide", handlePageHide);
			// 视频被移动到了 body，组件卸载时需要显式移除，避免残留。
			video?.remove();
		};
	});
</script>

<div class="music-dance">
	<button
		id="mzk-dance-control"
		class="dance-button"
		type="button"
		disabled={playing}
		onclick={startDance}
	>
		让网站变得通透
	</button>

	<audio bind:this={audio} preload="none"></audio>
	<video
		bind:this={video}
		class="floating-dance-video"
		class:visible={playing}
		style={`width:${CONFIG.videoWidth}px;height:${CONFIG.videoHeight}px;opacity:${CONFIG.videoOpacity};z-index:${CONFIG.videoZIndex};`}
		muted
		loop
		playsinline
		preload="metadata"
		aria-hidden="true"
	></video>

	{#if errorMessage}
		<span class="dance-error" role="status">
			{errorMessage}
		</span>
	{/if}
</div>

<style>
	.music-dance {
		display: flex;
		justify-content: center;
		align-items: center;
		width: 100%;
		margin: 24px auto;
		padding: 12px 16px;
		box-sizing: border-box;
	}

	.dance-button {
		border: 1px solid var(--line-color, #8884);
		border-radius: 999px;
		padding: 12px 24px;
		background: var(--card-bg, #fff);
		color: var(--primary, #6750a4);
		box-shadow: 0 4px 16px #0002;
		font-size: 14px;
		font-weight: 600;
		cursor: pointer;
		transition: opacity 0.2s;
	}

	.dance-button:hover {
		opacity: 0.82;
	}

	.dance-button:disabled {
		cursor: default;
		opacity: 0.7;
	}

	.dance-error {
		margin-left: 12px;
		color: #d44;
		font-size: 12px;
	}
	.floating-dance-video {
		position: fixed;
		left: 8px;
		top: 8px;
		object-fit: contain;
		visibility: hidden;
		pointer-events: none;
		border-radius: 12px;
		box-shadow: 0 4px 20px #0004;
		transition: opacity 0.3s ease;
		will-change: left, top;
	}

	.floating-dance-video.visible {
		visibility: visible;
	}

	@media (prefers-reduced-motion: reduce) {
		.floating-dance-video {
			transition: none;
		}
	}
</style>
