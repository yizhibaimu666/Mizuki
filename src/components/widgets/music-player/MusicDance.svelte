<script lang="ts">
	import { onMount } from "svelte";

	// ==================== 可编辑配置区 ====================
	const CONFIG = {
		musicUrl: "/music/dance.mp3", // 音乐文件路径
		// 悬浮视频配置
		videoSrc: "/video/dance.mp4", // 视频路径
		videoWidth: 240, // 视频宽度（px）
		videoHeight: 135, // 视频高度（px）
		videoSpeed: 35, // 移动速度（px/s）
		videoOpacity: 0.9, // 透明度（0～1）
		videoEdgePadding: 8, // 距离窗口边缘的间距（px）
		videoZIndex: 9999, // 悬浮层级
		videoTurnAngle: 55, // 碰边时随机偏转角度（度）

		flipDuration: 300, // 左右翻转运动时间，毫秒
		flipGap: 300, // 翻转停顿时间，毫秒

		stretchDuration: 300, // 上下拉伸运动时间，毫秒
		stretchGap: 300, // 拉伸停顿时间，毫秒

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
			"#sidebar-sticky",
			"#sidebar .sidebar-content > *",
			"#sidebar-sticky .sidebar-content > *",
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
	function resizeFloatingVideo() {
		if (!video) return;

		const padding = CONFIG.videoEdgePadding;
		const maxX = Math.max(
			padding,
			window.innerWidth - CONFIG.videoWidth - padding,
		);
		const maxY = Math.max(
			padding,
			window.innerHeight - CONFIG.videoHeight - padding,
		);

		videoX = Math.min(videoX, maxX);
		videoY = Math.min(videoY, maxY);

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
		video.src = CONFIG.videoSrc;

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
