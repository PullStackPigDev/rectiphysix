<script lang="ts">
	import { T } from "@threlte/core";
	import { Gizmo, Grid, interactivity, OrbitControls } from "@threlte/extras";
	// import { TransformControls } from "@threlte/extras";
	import RObject from "./RObject";
	import { Collider, Debug, RigidBody, useRapier } from "@threlte/rapier";
	import { Pane, Button, Point, Checkbox, Text, type PointValue2d } from "svelte-tweakpane-ui";
	import { globalState, physicsActive, resetRotation, type Project, type KeyedItem, savedState } from "$lib/stores";
	import Root from "./Root.svelte";
	import { createEventDispatcher, onDestroy, onMount, SvelteComponent } from "svelte";

	export let project: string;

	$: {
		if (project) {
			objects = [];
			selected = "";
		}
	}

	let objects: (KeyedItem & { key: string })[] = [];
	let objectComponents: SvelteComponent[] = [];
	let selected = "";

	// default value
	let gravity = {
		x: 0,
		y: -9.807,
		z: 0,
	};

	let mounted = false;

	interactivity();

	const { world } = useRapier();

	const dispatch = createEventDispatcher<{
		restart: void;
	}>();

	physicsActive.subscribe((v) => {
		// world.gravity = v ? gravity : { x: 0, y: 0, z: 0 };
		world.gravity = v ? gravity : { x: 0, y: 0, z: 0 };
	});

	resetRotation.subscribe((v) => {
		if (v) resetRotation.set(false);
	});

	onMount(() => {
		mounted = true;
		// this shouldn't be possible
		if (!$globalState) return;
		gravity = $globalState[project].gravity;
		objects = Object.entries($globalState[project])
			.filter((o) => o[0] !== "gravity" && o[0] !== "position")
			.map(([k, v]) => ({
				key: k,
				...(v as KeyedItem),
			}));
	});

	onDestroy(() => {
		objectComponents = [];
	});

	function newObject() {
		if ($globalState === null) {
			alert("loading, please wait");
			return;
		}
		const newObj = prompt("Please provide an unique id");
		if (newObj === "gravity") {
			alert('Object cannot be named "gravity"!');
			return;
		}
		if (newObj) {
			objects = [
				...Object.entries($globalState[project])
					.filter((o) => o[0] !== "gravity" && o[0] !== "position")
					.map(([k, v]) => ({
						key: k,
						...(v as KeyedItem),
					})),
				{
					key: newObj,
					position: {
						x: 0,
						y: 0,
						z: 0,
					},
					rotation: [0, 0, 0, 1],
					additionalMass: 0,
					resetOnGround: false,
					initialVelocity: [0, 0, 0],
					initialForce: [0, 0, 0],
					initialImpulse: [0, 0, 0],
					timer: 0,
					geometryType: "Box",
					geometryArgs: {
						Box: [1, 1, 1],
						Capsule: [1, 1, 16, 16],
						Sphere: [1, 16, 16],
						Cylinder: [1, 1, 2, 16],
					},
					color: "#ffffff",
				},
			];
		}
	}

	function save() {
		objectComponents.forEach((o) => {
			o.saveObj();
		});
		savedState.update((s) => {
			if (s === null) return null;
			const newState = {
				...s,
				[project]: {
					...s[project],
					gravity,
				} as Project,
			};
			return newState;
		});
	}

	$: {
		// console.log(project);
		globalState.update((s) => {
			if (s === null) return null;
			// console.log({
			// 	...s,
			// 	[project]: {
			// 		...s[project],
			// 		gravity,
			// 	} as Project,
			// });
			return {
				...s,
				[project]: {
					...s[project],
					gravity,
				} as Project,
			};
		});
	}

	$: {
		if (mounted) localStorage.setItem("rectiphysix-state", JSON.stringify($savedState));
	}

	let dummy: boolean = false;

	let gridSize: PointValue2d & [number, number] = [20, 20];
</script>

<svelte:window on:keydown={(e) => e.key === "Escape" && (selected = "")} />

{#if mounted}
	<Root>
		<Pane title="Rectiphysix management panel" position="fixed">
			<Text value={`Project: ${project}`} disabled />
			<Checkbox label="Physics enabled" bind:value={dummy} on:change={(e) => physicsActive.set(e.detail.value)} />

			<Point bind:value={gridSize} label="Grid helper size" disabled={$physicsActive} />
			<Point bind:value={gravity} label="Gravity values" disabled={$physicsActive} />
			<Button
				title="Reset all rotation"
				on:click={() => {
					resetRotation.update((v) => !v);
				}}
				disabled={$physicsActive}
			/>
			<Button title="Save" on:click={save} />
			<Button
				title="Load last saved"
				on:click={() => {
					dispatch("restart");
				}}
				disabled={$physicsActive}
			/>
			<Button
				title="Load last saved"
				on:click={() => {
					dispatch("restart");
				}}
				disabled={$physicsActive}
			/>
			<Button
				title="Add object"
				on:click={newObject}
				disabled={$physicsActive}
			/>
			<Button title="Add object" on:click={newObject} />
		</Pane>
	</Root>
{/if}

<T.PerspectiveCamera makeDefault position={[10, 10, -10]} fov={90}>
	<OrbitControls enableZoom enableDamping />
</T.PerspectiveCamera>

<T.AmbientLight intensity={1} />

<!-- Loaded from localstorage -->
{#if $globalState !== null}
	{#each objects as obj, ind (obj.key)}
		<RObject.View
			bind:this={objectComponents[ind]}
			key={obj.key}
			initialLinvel={obj.initialVelocity}
			time={obj.timer}
			additionalMass={obj.additionalMass}
			resetOnGround={obj.resetOnGround}
			color={obj.color}
			positionX={obj.position.x}
			positionY={obj.position.y}
			positionZ={obj.position.z}
			exportedRotation={obj.rotation}
			geometryType={obj.geometryType}
			args={obj.geometryArgs}
			selected={selected === obj.key}
			initialForce={obj.initialForce}
			initialImpulse={obj.initialImpulse}
			{project}
			on:select={() => (selected = obj.key)}
			on:delete={() => {
				objects = objects.filter((k) => k.key !== obj.key);
				localStorage.removeItem(
					`svelte-tweakpane-ui-draggable-position-${obj.key}`,
				);
			}}
		/>
	{/each}
{/if}

<!-- Debug -->
<Debug />

<T.AxesHelper position={[0, 0.01, 0]} args={[3]} renderOrder={1} />
<Gizmo />

<!-- Ground -->
<Grid {gridSize} cellColor="white" sectionColor="white" sectionSize={10} />
<RigidBody
	type="fixed"
	userData={{
		name: "ground",
	}}
>
	<Collider shape="cuboid" args={[10, 0, 10]} friction={0} restitution={0} />
</RigidBody>
