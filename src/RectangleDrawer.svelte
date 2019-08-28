<script>

	let i = 0;
	let numRects = 1;
	let undoStack = [[]];
	let rects = [];
	let selected;
	let adjusting = false;
	let adjusted = false;

	function handleClick(event) {
		if (adjusting) {
			adjusting = false;

			// if rect was adjusted,
			// push to the stack
			if (adjusted) push();
			return;
		}

		const rect = {
			name: `box-${numRects}`,
			x: event.clientX,
			y: event.clientY,
			width: 50,
			height: 50,
		};

		rects = rects.concat(rect);
		selected = rect;

		push();
		numRects++;
	}

	function adjust(event) {
		selected.width = +event.target.value;
		rects = rects;
		adjusted = true;
	}

	function select(rect, event) {
		if (!adjusting) {
			event.stopPropagation();
			selected = rect;
		}
	}

	function push() {
		const newUndoStack = undoStack.slice(0, ++i);
		newUndoStack.push(clone(rects));
		undoStack = newUndoStack;
	}

	function travel(d) {
		rects = clone(undoStack[i += d]);
		adjusting = false;
	}

	function clone(rects) {
		return rects.map(({ name, x, y, width, height }) => ({ name, x, y, width, height }));
	}
</script>

<style>
	.controls {
		position: relative;
		width: 100%;
		text-align: center;
	}

	svg {
		background-color: #eee;
		position: absolute;
    	width: 100%;
		height: 100%;
	}

	rect {
		stroke: black;
	}

	.adjuster {
		position: absolute;
		width: 80%;
		top: 10%;
		left: 50%;
		transform: translate(-50%,-50%);
		padding: 1em;
		text-align: center;
		background-color: rgba(255,255,255,0.7);
		border-radius: 4px;
	}

	input[type='range'] {
		width: 100%;
	}
</style>

<div>
<div class="controls">
	<button on:click="{() => travel(-1)}" disabled="{i === 0}">undo</button>
	<button on:click="{() => travel(+1)}" disabled="{i === undoStack.length -1}">redo</button>
</div>

<svg on:click={handleClick} >
	{#each rects as rect}
		<rect 
			x={rect.x} 
			y={rect.y} 
			width={rect.width}
			height={rect.height} 
			on:click="{event => select(rect, event)}"
			on:contextmenu|stopPropagation|preventDefault="{() => {
				adjusting = !adjusting;
				if (adjusting) selected = rect;
			}}"
			fill="{rect === selected ? '#ccc': 'white'}"
		/>
	{/each}
</svg>

{#if adjusting}
	<div class="adjuster">
		<p>adjust width of {selected.name} at {selected.x}, {selected.y}, {selected.width}, {selected.height}
		</p>
		<input type="range" value={selected.width} on:input={adjust}>
	</div>
{/if}
</div>
