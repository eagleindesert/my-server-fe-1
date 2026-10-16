<script lang="ts">
	import { onMount } from 'svelte';

	let canvas: HTMLCanvasElement;
	let width: number;
	let height: number;
	let cubes: RainbowCube3D[] = [];
	let mouse = { x: undefined as number | undefined, y: undefined as number | undefined };

	class POINT3D {
		x: number;
		y: number;
		z: number;
		constructor(x: number, y: number, z: number) {
			this.x = x;
			this.y = y;
			this.z = z;
		}
	}

	class RainbowCube3D {
		size: number;
		x: number;
		y: number;
		z: number;
		vx: number;
		vy: number;
		rx: number;
		ry: number;
		vrx: number;
		vry: number;
		hueOffset: number;
		vertices: POINT3D[];
		faces: number[][];

		constructor() {
			this.vertices = [
				new POINT3D(-1, -1, -1),
				new POINT3D(1, -1, -1),
				new POINT3D(1, 1, -1),
				new POINT3D(-1, 1, -1),
				new POINT3D(-1, -1, 1),
				new POINT3D(1, -1, 1),
				new POINT3D(1, 1, 1),
				new POINT3D(-1, 1, 1)
			];

			this.faces = [
				[0, 1, 2, 3],
				[1, 5, 6, 2],
				[5, 4, 7, 6],
				[4, 0, 3, 7],
				[3, 2, 6, 7],
				[4, 5, 1, 0]
			];

			this.size = Math.random() * 50 + 30;
			this.x = Math.random() * width;
			this.y = Math.random() * height;
			this.z = Math.random() * 0.5 + 0.5;
			this.vx = (Math.random() - 0.5) * 0.5;
			this.vy = (Math.random() - 0.5) * 0.5;
			this.rx = Math.random() * Math.PI * 2;
			this.ry = Math.random() * Math.PI * 2;
			this.vrx = (Math.random() - 0.5) * 0.015;
			this.vry = (Math.random() - 0.5) * 0.015;
			this.hueOffset = Math.random() * 360;
		}

		project(x: number, y: number, z: number) {
			return {
				x: this.x + x * this.size,
				y: this.y + y * this.size
			};
		}

		update() {
			this.x += this.vx * this.z;
			this.y += this.vy * this.z;
			this.rx += this.vrx;
			this.ry += this.vry;

			if (mouse.x !== undefined && mouse.y !== undefined) {
				const dx = this.x - mouse.x;
				const dy = this.y - mouse.y;
				const dist = Math.sqrt(dx * dx + dy * dy);
				const radius = 300;

				if (dist < radius) {
					const force = (radius - dist) / radius;
					this.x += (dx / dist) * force * 2;
					this.y += (dy / dist) * force * 2;
					this.rx += this.vrx * 2;
					this.ry += this.vry * 2;
				}
			}

			const margin = 150;
			if (this.x < -margin) this.x = width + margin;
			if (this.x > width + margin) this.x = -margin;
			if (this.y < -margin) this.y = height + margin;
			if (this.y > height + margin) this.y = -margin;
		}

		draw(ctx: CanvasRenderingContext2D) {
			let transformedVertices = this.vertices.map((v) => {
				let x1 = v.x * Math.cos(this.ry) - v.z * Math.sin(this.ry);
				let z1 = v.x * Math.sin(this.ry) + v.z * Math.cos(this.ry);
				let y1 = v.y * Math.cos(this.rx) - z1 * Math.sin(this.rx);
				let z2 = v.y * Math.sin(this.rx) + z1 * Math.cos(this.rx);
				return { x: x1, y: y1, z: z2 };
			});

			let projectedPoints = transformedVertices.map((v) => this.project(v.x, v.y, v.z));

			const lightDir = { x: 0.5, y: -0.8, z: -1 };
			const len = Math.sqrt(lightDir.x ** 2 + lightDir.y ** 2 + lightDir.z ** 2);
			lightDir.x /= len;
			lightDir.y /= len;
			lightDir.z /= len;

			ctx.lineJoin = 'round';
			ctx.lineCap = 'round';

			this.faces.forEach((face) => {
				const v0 = transformedVertices[face[0]];
				const v1 = transformedVertices[face[1]];
				const v2 = transformedVertices[face[2]];

				const A = { x: v1.x - v0.x, y: v1.y - v0.y, z: v1.z - v0.z };
				const B = { x: v2.x - v0.x, y: v2.y - v0.y, z: v2.z - v0.z };

				const nx = A.y * B.z - A.z * B.y;
				const ny = A.z * B.x - A.x * B.z;
				const nz = A.x * B.y - A.y * B.x;

				const nLen = Math.sqrt(nx * nx + ny * ny + nz * nz);
				const normal = { x: nx / nLen, y: ny / nLen, z: nz / nLen };

				let intensity = normal.x * lightDir.x + normal.y * lightDir.y + normal.z * lightDir.z;
				let brightness = Math.max(0.2, intensity);

				const alpha = 0.6 + brightness * 0.3;
				const lightness = 75 + brightness * 15;

				const p = face.map((idx) => projectedPoints[idx]);

				const grad = ctx.createLinearGradient(p[0].x, p[0].y, p[2].x, p[2].y);
				grad.addColorStop(0.0, `hsla(340, 100%, ${lightness}%, ${alpha})`);
				grad.addColorStop(0.3, `hsla(40,  100%, ${lightness}%, ${alpha})`);
				grad.addColorStop(0.6, `hsla(200, 100%, ${lightness}%, ${alpha})`);
				grad.addColorStop(1.0, `hsla(260, 100%, ${lightness}%, ${alpha})`);

				ctx.beginPath();
				ctx.moveTo(p[0].x, p[0].y);
				ctx.lineTo(p[1].x, p[1].y);
				ctx.lineTo(p[2].x, p[2].y);
				ctx.lineTo(p[3].x, p[3].y);
				ctx.closePath();

				ctx.fillStyle = grad;
				ctx.strokeStyle = grad;
				ctx.lineWidth = 32;

				ctx.stroke();
				ctx.fill();

				if (intensity > 0.5) {
					const glossOpacity = (intensity - 0.5) * 0.9;
					const glossGrad = ctx.createLinearGradient(p[0].x, p[0].y, p[2].x, p[2].y);
					glossGrad.addColorStop(0, `rgba(255, 255, 255, ${glossOpacity})`);
					glossGrad.addColorStop(0.6, `rgba(255, 255, 255, 0)`);

					ctx.fillStyle = glossGrad;
					ctx.fill();
				}
			});
		}
	}

	function initCubes() {
		cubes = [];
		const numCubes = Math.floor((width * height) / 100000);
		for (let i = 0; i < numCubes; i++) {
			cubes.push(new RainbowCube3D());
		}
	}

	function resize() {
		const dpr = Math.max(window.devicePixelRatio || 1, 2);
		width = window.innerWidth;
		height = window.innerHeight;
		canvas.width = width * dpr;
		canvas.height = height * dpr;
		const ctx = canvas.getContext('2d');
		if (ctx) {
			ctx.scale(dpr, dpr);
		}
		initCubes();
	}

	function animate() {
		const ctx = canvas.getContext('2d');
		if (!ctx) return;

		ctx.clearRect(0, 0, width, height);

		for (let i = 0; i < cubes.length; i++) {
			cubes[i].update();
			cubes[i].draw(ctx);
		}

		requestAnimationFrame(animate);
	}

	onMount(() => {
		resize();
		animate();

		const handleResize = () => resize();
		const handleMouseMove = (e: MouseEvent) => {
			mouse.x = e.x;
			mouse.y = e.y;
		};
		const handleMouseLeave = () => {
			mouse.x = undefined;
			mouse.y = undefined;
		};

		window.addEventListener('resize', handleResize);
		window.addEventListener('mousemove', handleMouseMove);
		window.addEventListener('mouseleave', handleMouseLeave);

		return () => {
			window.removeEventListener('resize', handleResize);
			window.removeEventListener('mousemove', handleMouseMove);
			window.removeEventListener('mouseleave', handleMouseLeave);
		};
	});
</script>

<canvas id="bgCanvas" bind:this={canvas}></canvas>

<div class="container">
	<h1>Tory's Server</h1>
	<p>Welcome to Tory Space!</p>
	<div class="contact-links">
		<a class="email-link" href="mailto:toridoremi@naver.com">
			<svg
				width="22"
				height="22"
				viewBox="0 0 24 24"
				fill="none"
				stroke="currentColor"
				stroke-width="1.7"
				stroke-linecap="round"
				stroke-linejoin="round"
				aria-hidden="true"
			>
				<rect x="3" y="5" width="18" height="14" rx="3" />
				<path d="m3 7 9 6 9-6" />
			</svg>
			<span>toridoremi@naver.com</span>
		</a>
		<a
			class="github-link"
			href="https://github.com/eagleindesert"
			target="_blank"
			rel="noopener noreferrer"
			aria-label="GitHub 프로필 (새 탭에서 열림)"
		>
			<svg width="20" height="20" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true">
				<path
					d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82A7.65 7.65 0 0 1 8 3.86c.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0 0 16 8c0-4.42-3.58-8-8-8Z"
				/>
			</svg>
			<span>GitHub</span>
			<svg
				width="18"
				height="18"
				viewBox="0 0 24 24"
				fill="none"
				stroke="currentColor"
				stroke-width="1.7"
				stroke-linecap="round"
				stroke-linejoin="round"
				aria-hidden="true"
			>
				<path d="M7 17 17 7M7 7h10v10" />
			</svg>
		</a>
	</div>
</div>

<footer>&copy; 2026 Tory's Server. All Rights Reserved.</footer>

<style>
	@import url('https://fonts.googleapis.com/css2?family=Chakra+Petch:wght@700&display=swap');
	@import url('https://cdn.jsdelivr.net/gh/orioncactus/pretendard/dist/web/static/pretendard.css');

	:global(body) {
		margin: 0;
		padding: 0;
		box-sizing: border-box;
		font-family:
			'Pretendard',
			-apple-system,
			BlinkMacSystemFont,
			system-ui,
			Roboto,
			sans-serif;
		background-color: #f8f5f0;
		color: #2d3436;
		height: 100vh;
		overflow: hidden;
	}

	#bgCanvas {
		position: fixed;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		z-index: 0;
	}

	.container {
		box-sizing: border-box;
		position: fixed;
		top: 50%;
		left: 50%;
		transform: translate(-50%, -50%);
		z-index: 10;
		background: linear-gradient(135deg, rgba(255, 255, 255, 0.22), rgba(255, 255, 255, 0.08));
		backdrop-filter: blur(16px) saturate(115%);
		-webkit-backdrop-filter: blur(16px) saturate(115%);
		padding: 4.5rem 4rem;
		border-radius: 28px;
		border: 1px solid rgba(255, 255, 255, 0.3);
		text-align: center;
		max-width: 680px;
		width: 90%;
		box-shadow:
			0 8px 32px rgba(45, 52, 54, 0.025),
			0 1px 0 rgba(255, 255, 255, 0.3) inset;
		transition:
			transform 0.3s ease,
			box-shadow 0.3s ease;
	}

	.container:hover {
		transform: translate(-50%, -50%) translateY(-2px);
		box-shadow:
			0 12px 36px rgba(45, 52, 54, 0.04),
			0 1px 0 rgba(255, 255, 255, 0.4) inset;
	}

	h1 {
		font-family: 'Chakra Petch', 'Pretendard', sans-serif;
		font-size: 4.5rem;
		font-weight: 700;
		margin: 0 0 1rem;
		letter-spacing: -0.04em;
		color: #ffffff;
		-webkit-text-stroke: 2px #202124;
		paint-order: stroke fill;
		text-shadow:
			1px 1px 0 #202124,
			2px 2px 0 #202124,
			3px 3px 0 #202124,
			0 6px 12px rgba(32, 33, 36, 0.14);
	}

	p {
		font-size: 1.35rem;
		line-height: 1.6;
		color: #636e72;
		margin: 0 0 2rem;
		font-weight: 500;
	}

	.contact-links {
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 1.25rem;
	}

	.email-link {
		display: inline-flex;
		align-items: center;
		justify-content: center;
		gap: 0.75rem;
		box-sizing: border-box;
		max-width: 100%;
		padding: 1rem 1.5rem;
		border: 1px solid rgba(108, 92, 231, 0.2);
		border-radius: 16px;
		background: linear-gradient(135deg, rgba(108, 92, 231, 0.1), rgba(253, 121, 168, 0.1));
		color: #6550ce;
		font-size: 1.15rem;
		font-weight: 650;
		text-decoration: none;
		transition:
			background-color 0.2s ease,
			box-shadow 0.2s ease;
	}

	.email-link svg {
		flex-shrink: 0;
	}

	.email-link span {
		min-width: 0;
		overflow-wrap: anywhere;
	}

	.email-link:hover {
		background-color: rgba(108, 92, 231, 0.08);
		box-shadow: 0 6px 20px rgba(108, 92, 231, 0.12);
	}

	.email-link:focus-visible,
	.github-link:focus-visible {
		outline: 3px solid #6c5ce7;
		outline-offset: 4px;
	}

	.github-link {
		display: inline-flex;
		align-items: center;
		gap: 0.375rem;
		min-height: 44px;
		padding: 0 0.75rem;
		border-radius: 8px;
		color: #636e72;
		font-size: 1rem;
		font-weight: 600;
		text-decoration: none;
		transition: color 0.2s ease;
	}

	.github-link:hover {
		color: #6550ce;
		text-decoration: underline;
		text-underline-offset: 4px;
	}

	footer {
		position: fixed;
		right: 1rem;
		bottom: calc(1.5rem + env(safe-area-inset-bottom, 0px));
		left: 1rem;
		z-index: 10;
		text-align: center;
		font-size: 0.9rem;
		color: #747d82;
		font-weight: 600;
	}

	@media (max-width: 768px) {
		h1 {
			font-size: 3rem;
			-webkit-text-stroke-width: 1.5px;
		}
		.container {
			padding: 3rem 2rem;
		}
		.email-link {
			padding: 0.875rem 1.125rem;
			font-size: 1rem;
		}
		footer {
			font-size: 0.75rem;
		}
	}
</style>
