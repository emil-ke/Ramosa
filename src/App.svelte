<script lang="ts">
	import {
		SvelteFlow,
		Controls,
		Background,
		MiniMap,
		type Node,
		type Edge,
		type Connection,
	} from "@xyflow/svelte";
	import { onMount } from "svelte";
	import "@xyflow/svelte/dist/style.css";
	import TextUpdaterNode from "./TextUpdaterNode.svelte";

	const nodeTypes = { textUpdater: TextUpdaterNode };

	let nodeCounter = 0;
	let leftRight = 1; // kind of a stupid variable for adding nodes left/right of current node (is either 1 or -1)
	let wiping = false;

	// plain reactive arrays work better, it seems
	let nodes: Node[] = [
		{
			id: "1",
			type: "textUpdater",
			data: { name: "start", text: "Did you find the key yet?", choices: [] },
			position: { x: 0, y: 0 },
		},
	];
	let edges: Edge[] = [];

	function recomputeCounter() {
		const max = nodes
			.map((n) => Number(n.id))
			.filter((v) => !Number.isNaN(v))
			.reduce((a, b) => Math.max(a, b), 0);
		nodeCounter = Math.max(nodeCounter, max);
	}

	recomputeCounter();

	// autosave/restore
	onMount(() => {
		const raw = localStorage.getItem("dialogue_autosave");
		if (!raw) return;
		try {
			const parsed = JSON.parse(raw);
			if (Array.isArray(parsed.nodes) && Array.isArray(parsed.edges)) {
				// normalize ids to strings and set nodes/edges as reactive arrays
				nodes = parsed.nodes.map((n: any, i: number) => ({
					...n,
					id: String(n.id ?? i + 1),
				}));
				edges = parsed.edges.map((e: any) => ({ ...e, id: String(e.id) }));
				recomputeCounter();
			}
		} catch (e) {
			console.warn("Failed to parse autosave:", e);
		}
	});

	// autosave whenever nodes/edges change
	$: {
		try {
			localStorage.setItem(
				"dialogue_autosave",
				JSON.stringify({ nodes, edges }),
			);
		} catch (e) {
			console.warn("Autosave failed", e);
		}
	}

	function getNextId() {
		nodeCounter++;
		return String(nodeCounter);
	}

	function onKeyDown(e: KeyboardEvent) {
		// ignore key events if the user is typing in an input or textarea
		const target = e.target as HTMLElement;
		if (target && (target.tagName === 'INPUT' || target.tagName === 'TEXTAREA' || target.isContentEditable)) {
			return;
		}

		if (e.key === 'n' || e.key === 'N') {
			e.preventDefault(); // only prevent default for this key
			addNode();
		}
	}

	function addNode() {
		const id = getNextId();
		leftRight *= -1;
		const newNode: Node = {
			id,
			type: "textUpdater",
			data: { name: `node_${id}`, text: "", choices: [] },
			position: { x: -170 * nodeCounter * leftRight, y: 300 * nodeCounter },
		};
		// reassign to trigger reactivity (important)
		nodes = [...nodes, newNode];
	}

	function onConnect(connection: Connection) {
		// always reassign edges (avoid .push)
		const newEdge: Edge = {
			id: `${connection.source}-${connection.target}`,
			source: connection.source!,
			target: connection.target!,
		};
		edges = [...edges, newEdge];
	}

	function exportDialogue() {
		const result: Record<string, any> = { nodes: {} };
		for (const n of nodes) {
			const name = n.data?.name || n.id;
			const nodeObj: any = { text: n.data?.text || "", choices: [] };
			for (const c of n.data?.choices ?? []) {
				const choice: any = { text: c.text || "" };
				if (c.next) choice.next = c.next;
				if (Array.isArray(c.traits) && c.traits.length) {
					choice.traits = c.traits.map((t: any) =>
						!t.amount || t.amount === 1
							? t.name
							: { name: t.name, amount: t.amount },
					);
				}
				if (Array.isArray(c.conditions) && c.conditions.length)
					choice.conditions = c.conditions;
				nodeObj.choices.push(choice);
			}
			result.nodes[name] = nodeObj;
		}
		const json = JSON.stringify(result, null, 4);
		const blob = new Blob([json], { type: "application/json" });
		const url = URL.createObjectURL(blob);
		const a = document.createElement("a");
		a.href = url;
		a.download = "dialogue.json";
		a.click();
		URL.revokeObjectURL(url);
	}

	function importDialogue() {
		const input = document.createElement("input");
		input.type = "file";
		input.accept = "application/json";
		input.onchange = async () => {
			const file = input.files?.[0];
			if (!file) return;
			const text = await file.text();
			let json: any;
			try {
				json = JSON.parse(text);
			} catch {
				alert("Invalid JSON");
				return;
			}
			if (!json.nodes || typeof json.nodes !== "object") {
				alert("JSON missing 'nodes' object");
				return;
			}
			// wipie wipe
			nodes = [];
			edges = [];
			const names = Object.keys(json.nodes);
			const idMap: Record<string, string> = {};
			names.forEach((name, i) => {
				const raw = json.nodes[name];
				const id = String(i + 1);
				idMap[name] = id;
				nodes = [
					...nodes,
					{
						id,
						type: "textUpdater",
						data: {
							name,
							text: raw.text || "",
							choices: (raw.choices ?? []).map((c: any) => {
								const choice: any = {
									text: c.text || "",
									next: c.next || "",
									traits: [],
									conditions: [],
								};
								if (Array.isArray(c.traits)) {
									choice.traits = c.traits.map((t: any) =>
										typeof t === "string"
											? { name: t, amount: 1 }
											: { name: t.name || "", amount: t.amount ?? 1 },
									);
								}
								if (Array.isArray(c.conditions))
									choice.conditions = c.conditions;
								return choice;
							}),
						},
						position: { x: 500 * (i % 2 === 0 ? 1 : -1), y: 400 * i },
					},
				];
			});
			names.forEach((name) => {
				const nodeId = idMap[name];
				const node = json.nodes[name];
				if (!node?.choices) return;
				node.choices.forEach((c: any) => {
					if (!c.next) return;
					const targetId = idMap[c.next];
					if (!targetId) return;
					edges = [
						...edges,
						{ id: `${nodeId}-${targetId}`, source: nodeId, target: targetId },
					];
				});
			});
			recomputeCounter();
		};
		input.click();
	}

	function confirmWipe() {
		if (!wiping) {
			wiping = true;
			if (confirm("This will ERASE the entire dialogue. Continue?")) {
				doWipe();
			}
			wiping = false;
		}
	}
	function doWipe() {
		localStorage.removeItem("dialogue_autosave");
		nodeCounter = 0;
		leftRight = 1;
		nodes = [
			{
				id: "1",
				type: "textUpdater",
				data: { name: "start", text: "", choices: [] },
				position: { x: 0, y: 0 },
			},
		];
		edges = [];
		recomputeCounter();
	}
</script>

<div style="height: 100vh; width: 100vw; position: relative;">
	<div class="btns">
		<button onclick={addNode} class="top_btn" style="background: #3b82f6;"
			>+ Add Node</button
		>
		<button
			onclick={exportDialogue}
			class="top_btn"
			style="background: #10b981;">Export JSON</button
		>
		<button
			onclick={importDialogue}
			class="top_btn"
			style="background: #f59e0b;">Import JSON</button
		>
		<button onclick={confirmWipe} class="top_btn" style="background: #ef4444;"
			>New Dialogue</button
		>
	</div>

	<SvelteFlow bind:nodes bind:edges {nodeTypes} onconnect={onConnect} fitView>
		<Controls orientation="horizontal" />
		<Background bgColor="#111" />
		<MiniMap bgColor="#000" nodeColor="#333" maskColor="#222"/>
	</SvelteFlow>
</div>

<style>
	.btns {
		position: absolute;
		top: 10px;
		left: 10px;
		z-index: 100;
	}
	.top_btn {
		padding: 0.5rem 1.6rem;
		font-size: 0.9rem;
		color: white;
		border-radius: 8px;
		border: none;
		cursor: pointer;
		box-shadow: 0 2px 5px rgba(0, 0, 0, 0.15);
		margin-right: 0.5rem;
	}
</style>

<svelte:window on:keydown={onKeyDown} />
