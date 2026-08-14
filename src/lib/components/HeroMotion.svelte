<script lang="ts">
	import { onMount } from 'svelte';

	let canvas: HTMLCanvasElement;

	onMount(() => {
		let frame = 0;
		let cleanup: (() => void) | undefined;
		let active = true;

		const reducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)');
		const compactViewport = window.matchMedia('(max-width: 767px)');

		if (reducedMotion.matches || compactViewport.matches) return;

		void import('three').then((THREE) => {
			if (!active) return;

			const scene = new THREE.Scene();
			const camera = new THREE.PerspectiveCamera(32, 1, 0.1, 100);
			camera.position.set(0, 0.25, 8.4);

			const renderer = new THREE.WebGLRenderer({
				canvas,
				alpha: true,
				antialias: true,
				powerPreference: 'low-power'
			});
			renderer.setPixelRatio(Math.min(window.devicePixelRatio, 1.5));

			const sculpture = new THREE.Group();
			sculpture.rotation.z = -0.12;
			scene.add(sculpture);

			const wireGeometry = new THREE.IcosahedronGeometry(1.7, 2);
			const wireMaterial = new THREE.MeshBasicMaterial({
				color: 0xd97724,
				wireframe: true,
				transparent: true,
				opacity: 0.28
			});
			const wire = new THREE.Mesh(wireGeometry, wireMaterial);
			sculpture.add(wire);

			const coreGeometry = new THREE.IcosahedronGeometry(1.42, 1);
			const coreMaterial = new THREE.MeshBasicMaterial({
				color: 0xf0a35a,
				wireframe: true,
				transparent: true,
				opacity: 0.22
			});
			const core = new THREE.Mesh(coreGeometry, coreMaterial);
			sculpture.add(core);

			const ringGeometries: THREE.BufferGeometry[] = [];
			const ringMaterials: THREE.LineBasicMaterial[] = [];
			const rings: THREE.LineLoop[] = [];
			for (const config of [
				{ radius: 2.15, tilt: 0.35, rotate: 0.2, opacity: 0.5 },
				{ radius: 2.45, tilt: -0.65, rotate: -0.65, opacity: 0.3 },
				{ radius: 2.7, tilt: 1.05, rotate: 0.9, opacity: 0.18 }
			]) {
				const curve = new THREE.EllipseCurve(0, 0, config.radius, config.radius * 0.34);
				const geometry = new THREE.BufferGeometry().setFromPoints(curve.getPoints(96));
				const material = new THREE.LineBasicMaterial({
					color: 0xe4a33b,
					transparent: true,
					opacity: config.opacity
				});
				const ring = new THREE.LineLoop(geometry, material);
				ring.rotation.set(config.tilt, config.rotate, 0);
				sculpture.add(ring);
				ringGeometries.push(geometry);
				ringMaterials.push(material);
				rings.push(ring);
			}

			const nodePositions = new Float32Array(64 * 3);
			const goldenAngle = Math.PI * (3 - Math.sqrt(5));
			for (let i = 0; i < 64; i += 1) {
				const y = 1 - (i / 63) * 2;
				const radius = Math.sqrt(1 - y * y) * 2.15;
				const angle = goldenAngle * i;
				nodePositions[i * 3] = Math.cos(angle) * radius;
				nodePositions[i * 3 + 1] = y * 2.15;
				nodePositions[i * 3 + 2] = Math.sin(angle) * radius;
			}
			const nodeGeometry = new THREE.BufferGeometry();
			nodeGeometry.setAttribute('position', new THREE.BufferAttribute(nodePositions, 3));
			const nodeMaterial = new THREE.PointsMaterial({
				color: 0xf4c27d,
				size: 0.045,
				transparent: true,
				opacity: 0.72,
				depthWrite: false
			});
			const nodes = new THREE.Points(nodeGeometry, nodeMaterial);
			sculpture.add(nodes);

			const pointer = { x: 0, y: 0, targetX: 0, targetY: 0 };
			const onPointerMove = (event: PointerEvent) => {
				pointer.targetX = (event.clientX / window.innerWidth - 0.5) * 2;
				pointer.targetY = (event.clientY / window.innerHeight - 0.5) * 2;
			};
			const resize = () => {
				const { width, height } = canvas.getBoundingClientRect();
				if (!width || !height) return;
				camera.aspect = width / height;
				camera.updateProjectionMatrix();
				renderer.setSize(width, height, false);
			};
			const observer = new ResizeObserver(resize);
			observer.observe(canvas);
			window.addEventListener('pointermove', onPointerMove, { passive: true });
			resize();

			const startedAt = performance.now();
			const render = (time: number) => {
				if (!active) return;
				const elapsed = (time - startedAt) * 0.00018;
				pointer.x += (pointer.targetX - pointer.x) * 0.035;
				pointer.y += (pointer.targetY - pointer.y) * 0.035;
				sculpture.rotation.y = elapsed * 0.7 + pointer.x * 0.1;
				sculpture.rotation.x = Math.sin(elapsed * 0.8) * 0.06 + pointer.y * 0.05;
				wire.rotation.y = elapsed * 0.55;
				core.rotation.y = -elapsed * 0.85;
				core.rotation.x = elapsed * 0.35;
				rings[0].rotation.z = elapsed * 0.4;
				rings[1].rotation.z = -elapsed * 0.25;
				rings[2].rotation.z = elapsed * 0.18;
				nodes.rotation.y = -elapsed * 0.2;
				renderer.render(scene, camera);
				frame = requestAnimationFrame(render);
			};
			frame = requestAnimationFrame(render);

			cleanup = () => {
				cancelAnimationFrame(frame);
				observer.disconnect();
				window.removeEventListener('pointermove', onPointerMove);
				wireGeometry.dispose();
				wireMaterial.dispose();
				coreGeometry.dispose();
				coreMaterial.dispose();
				for (const geometry of ringGeometries) geometry.dispose();
				for (const material of ringMaterials) material.dispose();
				nodeGeometry.dispose();
				nodeMaterial.dispose();
				renderer.dispose();
			};
		});

		return () => {
			active = false;
			cleanup?.();
		};
	});
</script>

<canvas bind:this={canvas} class="hero-motion" aria-hidden="true"></canvas>

<style>
	.hero-motion {
		position: absolute;
		inset: 0;
		z-index: 0;
		width: 100%;
		height: 100%;
		pointer-events: none;
		opacity: 0.82;
		mix-blend-mode: multiply;
	}

	:global(:root.dark) .hero-motion {
		mix-blend-mode: screen;
		opacity: 0.56;
	}
</style>
