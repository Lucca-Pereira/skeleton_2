<script>
	import favicon from "$lib/assets/favicon.svg";
	import { onMount } from "svelte";
	import * as THREE from "three";
	import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";
	import { base } from '$app/paths';


	let { children } = $props();
	let canvasEl;
	let loadedModel = null;
	let dragButton = null;
	let lastX = 0;
	let lastY = 0;

	onMount(() => {
		// --- Scene ---
		const scene = new THREE.Scene();
		scene.background = new THREE.Color(0x414141); // light gray

		// --- Camera ---
		const aspect = window.innerWidth / window.innerHeight;
		const viewSize = 3;
		const camera = new THREE.OrthographicCamera(
			(-viewSize * aspect) / 2,
			(viewSize * aspect) / 2,
			viewSize / 2,
			-viewSize / 2,
			0.1,
			1000
		);
		camera.position.set(0, 0, 3);

		// --- Renderer ---
		const renderer = new THREE.WebGLRenderer({ canvas: canvasEl, antialias: true });
		renderer.setSize(window.innerWidth, window.innerHeight);

		renderer.setPixelRatio(window.devicePixelRatio);
		
		// Moviendo el raton hace tal
		
		canvasEl.addEventListener('pointerdown', (event) => {
			dragButton = event.button;
			lastX = event.clientX;
			lastY = event.clientY;	
			canvasEl.setPointerCapture(event.pointerId);
		});

		canvasEl.addEventListener('pointermove', (event) => {
			if (dragButton === null || !loadedModel) return;
			const deltaX = event.clientX - lastX;
			const deltaY = event.clientY - lastY;

			if (dragButton === 0) {
				// right-drag: rotate
				loadedModel.rotation.y += deltaX * 0.01;
				loadedModel.rotation.x += deltaY * 0.01;
			} else if (dragButton === 2) {
				// left-drag: pan
				loadedModel.position.x += deltaX * 0.0025 / camera.zoom;
				loadedModel.position.y -= deltaY * 0.0025 / camera.zoom;
			}

			lastX = event.clientX;
			lastY = event.clientY;
		});

		canvasEl.addEventListener('pointerup', () => {
			dragButton = null;
		});

		canvasEl.addEventListener('contextmenu', (event) => {
			event.preventDefault();
		});

		canvasEl.addEventListener('wheel', (event) => {
			event.preventDefault();

			const rect = canvasEl.getBoundingClientRect();
			const ndcX = ((event.clientX - rect.left) / rect.width) * 2 - 1;
			const ndcY = -(((event.clientY - rect.top) / rect.height) * 2 - 1);

			const halfWidth = (camera.right - camera.left) / 2;
			const halfHeight = (camera.top - camera.bottom) / 2;

			const oldZoom = camera.zoom;
			camera.zoom -= event.deltaY * 0.001;
			camera.zoom = Math.max(0.5, Math.min(5, camera.zoom));
			const newZoom = camera.zoom;

			camera.position.x += ndcX * halfWidth * (1 / oldZoom - 1 / newZoom);
			camera.position.y += ndcY * halfHeight * (1 / oldZoom - 1 / newZoom);

			camera.updateProjectionMatrix();
		}, { passive: false });

		window.addEventListener("resize", () => {
			const aspect = window.innerWidth / window.innerHeight;
			camera.left = (-viewSize * aspect) / 2;
			camera.right = (viewSize * aspect) / 2;
			camera.top = viewSize / 2;
			camera.bottom = -viewSize / 2;
			camera.updateProjectionMatrix();
			renderer.setSize(window.innerWidth, window.innerHeight);
		});

		// --- Lighting ---
		scene.add(new THREE.AmbientLight(0xffffff, 1.2));

		// Headlamp: attached to the camera, always lights whatever you're looking at
		const headLight = new THREE.DirectionalLight(0xffffff, 0.9);
		camera.add(headLight);
		camera.add(headLight.target);
		headLight.target.position.set(0, 0, -1);
		scene.add(camera);

		// --- Load the model ---
		let boneMesh = null;

		const loader = new GLTFLoader();
		loader.load(
			`${base}/model.glb`,
			(gltf) => {
				gltf.scene.traverse((child) => {
					if (child.isMesh) {
						if (!boneMesh) boneMesh = child;

						// Runtime patch: force emissive to black.
						// Z-Anatomy's exported material comes through with a
						// full-white emissive value regardless of what's set
						// in Blender's node group — overriding it here until
						// the actual export-side cause is tracked down.
						child.material.emissive.set(0x000000);
					}
				});

			const pivot = new THREE.Group();
			scene.add(pivot);
			pivot.add(gltf.scene);

			const box = new THREE.Box3().setFromObject(gltf.scene);
			const center = box.getCenter(new THREE.Vector3());
			const maxDim = Math.max(...box.getSize(new THREE.Vector3()).toArray());
			const scale = 2 / maxDim;

			gltf.scene.scale.setScalar(scale);
			gltf.scene.position.sub(center.multiplyScalar(scale)); // shifts geometry so it's centered *within* pivot

			loadedModel = pivot; // drag handlers now rotate/pan the pivot, not gltf.scene directly
			},
			undefined,
			(error) => console.error("Model failed to load:", error),
		);

		// --- Render loop ---
		function animate() {
			renderer.render(scene, camera);
		}
		renderer.setAnimationLoop(animate);
	});
</script>


<style>
	:global(html, body) {
		margin: 0;
		padding: 0;
		overflow: hidden;
		height: 100%;
	}

	canvas {
		display: block;
	}
</style>

<svelte:head>
	<link rel="icon" href={favicon} />
</svelte:head>

<canvas bind:this={canvasEl}></canvas>

{@render children()}