<script>
	import favicon from "$lib/assets/favicon.svg";
	import { onMount } from "svelte";
	import * as THREE from "three";
	import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";
	import { base } from '$app/paths';
	import { EffectComposer } from "three/examples/jsm/postprocessing/EffectComposer.js";
	import { RenderPass } from "three/examples/jsm/postprocessing/RenderPass.js";
	import { OutlinePass } from "three/examples/jsm/postprocessing/OutlinePass.js";
	import { OutputPass } from "three/examples/jsm/postprocessing/OutputPass.js";



	let { children } = $props();
	let canvasEl;
	let loadedModel = null;
	let dragButton = null;
	let lastX = 0;
	let lastY = 0;

	let hoveredName = $state(null);
	let tooltipX = $state(0);
	let tooltipY = $state(0);
	let insertionMeshes = [];

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
		
		//Composer addons
		
		//Add smoothness layer since its now passing through composer
		const renderTarget = new THREE.WebGLRenderTarget(
			window.innerWidth,
			window.innerHeight,
			{ samples: 4 }
		);

		//Composer pass "addPass"
		const composer = new EffectComposer(renderer, renderTarget);
		const renderPass = new RenderPass(scene, camera);
		composer.addPass(renderPass);

		//Raycaster
		const raycaster = new THREE.Raycaster();
		const pointer = new THREE.Vector2();

		//Edge creation
		const outlinePass = new OutlinePass(
			new THREE.Vector2(window.innerWidth, window.innerHeight),
			scene,
			camera
		);
		outlinePass.edgeStrength = 5;
		outlinePass.edgeGlow = 0.5;
		outlinePass.edgeThickness = 2;
		outlinePass.visibleEdgeColor.set(0xffff00);
		outlinePass.hiddenEdgeColor.set(0xffff00);
		composer.addPass(outlinePass);

		const outputPass = new OutputPass();
		composer.addPass(outputPass);

		// Moviendo el raton hace tal
		
		canvasEl.addEventListener('pointerdown', (event) => {
			dragButton = event.button;
			lastX = event.clientX;
			lastY = event.clientY;	
			canvasEl.setPointerCapture(event.pointerId);
		});

		canvasEl.addEventListener('pointermove', (event) => {
			if (!loadedModel) return;
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
			} else if (dragButton === null){
				const rect = canvasEl.getBoundingClientRect();
				pointer.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
				pointer.y = -(((event.clientY - rect.top) / rect.height) * 2 - 1);

				raycaster.setFromCamera(pointer, camera);


				const hits = raycaster.intersectObject(loadedModel, true);
				const insertionHit = hits.find((h) => insertionMeshes.includes(h.object));

				function baseName(name) {
					return name.replace(/_\d+$/, '');
				}

				if (insertionHit) {
					const parentBase = baseName(insertionHit.object.parent?.name || '');

					const trulyOccluded = hits.some(
						(h) => h.distance < insertionHit.distance - 0.0001 && baseName(h.object.name) !== parentBase
					);
					//console.log('Truly occluded:', trulyOccluded);


					if (!trulyOccluded) {
						outlinePass.selectedObjects = [insertionHit.object];
						hoveredName = insertionHit.object.name.replace(insertionPattern, '').replace(/_/g, ' ');
						tooltipX = event.clientX;
						tooltipY = event.clientY;
					} else {
						outlinePass.selectedObjects = [];
						hoveredName = null;
					}
				} else {
					outlinePass.selectedObjects = [];
					hoveredName = null;
				}
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
			camera.zoom -= event.deltaY * 0.0035;
			camera.zoom = Math.max(0.5, Math.min(20, camera.zoom));
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
			composer.setSize(window.innerWidth, window.innerHeight);
			renderTarget.setSize(window.innerWidth, window.innerHeight);
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

		const insertionPattern = /[oe]\d*[rl]$/;

		const loader = new GLTFLoader();
		loader.load(
			`${base}/model.glb`,
			(gltf) => {
				insertionMeshes = [];

				gltf.scene.traverse((child) => {
					if (child.isMesh) {
						child.material.emissive.set(0x000000);


						if (insertionPattern.test(child.name)) {
							insertionMeshes.push(child);
						} else if (!boneMesh) {
							boneMesh = child;
						}
					}
				});
				
				const insertionMaterial = boneMesh.material.clone();
				insertionMaterial.color.set(0xff4500);
				insertionMaterial.polygonOffset = true;
				insertionMaterial.polygonOffsetFactor = -4;
				insertionMaterial.polygonOffsetUnits = -4;

				insertionMeshes.forEach((mesh) => {
					mesh.material = insertionMaterial;
				});

			const pivot = new THREE.Group();
			scene.add(pivot);
			pivot.add(gltf.scene);

			const box = new THREE.Box3().setFromObject(gltf.scene);
			const center = box.getCenter(new THREE.Vector3());
			const maxDim = Math.max(...box.getSize(new THREE.Vector3()).toArray());
			const scale = 2 / maxDim;

			gltf.scene.scale.setScalar(scale);
			gltf.scene.position.sub(center.multiplyScalar(scale));

			loadedModel = pivot;
			},
			undefined,
			(error) => console.error("Model failed to load:", error),
		);

		// --- Render loop ---
		function animate() {
			composer.render();
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

	.tooltip {
		position: fixed;
		pointer-events: none;
		background: rgba(0, 0, 0, 0.85);
		color: white;
		padding: 4px 10px;
		border-radius: 4px;
		font-family: monospace;
		font-size: 13px;
		z-index: 10;
}
</style>

<svelte:head>
	<link rel="icon" href={favicon} />
</svelte:head>

<canvas bind:this={canvasEl}></canvas>

{#if hoveredName}
	<div class="tooltip" style="left: {tooltipX + 12}px; top: {tooltipY + 12}px;">
		{hoveredName}
	</div>
{/if}

{@render children()}