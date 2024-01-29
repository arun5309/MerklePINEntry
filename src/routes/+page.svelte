<script lang="ts">
	var redirect = false;
	// var url = '';
	const evtSource = new EventSource(`http://${import.meta.env.VITE_LOCAL_IP}:8000/stream`);
	evtSource.onmessage = function (event) {
		if (event.data) {
			redirect = true;
			window.location.replace(event.data);
			// window.location.href = event.data;
			// url = event.data; // Frame URL
			// url = `http://${import.meta.env.VITE_LOCAL_IP}:8000/${event.data}`; // Video URL
		} else {
			redirect = false;
		}
	};

	function rand_digit(): number {
		for (;;) {
			const array = new Uint8Array(1);
			self.crypto.getRandomValues(array);
			if (array[0] < 250) {
				return array[0] % 10;
			}
		}
	}

	function rand_digit_min_i(i: number): number {
		for (;;) {
			const j = rand_digit();
			if (j >= i) {
				return j;
			}
		}
	}

	function rand_perm(): Uint8Array {
		let array = new Uint8Array(10);
		for (var i = 0; i < 10; ++i) {
			array[i] = i;
		}
		for (i = 0; i < 9; ++i) {
			const j = rand_digit_min_i(i);
			const temp = array[i];
			array[i] = array[j];
			array[j] = temp;
		}
		return array;
	}

	function sub_mod100(a: number, b: number): number {
		return (((a - b) % 100) + 100) % 100;
	}

	function sub_mod10(a: number, b: number): number {
		return (((a - b) % 10) + 10) % 10;
	}

	type Puzzle = {
		sol: Uint8Array;
		q: Uint8Array;
	};

	function make_puzzle(): Puzzle {
		let puzzle_sol = rand_perm();
		let puzzle_q = new Uint8Array(10);
		for (var i = 0; i < 10; ++i) {
			puzzle_sol[i] = Number(String(puzzle_sol[i]) + String(rand_digit()));
			puzzle_q[i] = Number(String(rand_digit()) + String(rand_digit()));
		}
		return {
			sol: puzzle_sol,
			q: puzzle_q
		};
	}

	type PuzzleEntry = {
		sol: number;
		q: number;
	};

	type ZippedPuzzle = Array<PuzzleEntry>;

	function puzzle_zip(puzzle: Puzzle): ZippedPuzzle {
		let arr = [];
		for (var i = 0; i < 10; ++i) {
			arr.push({
				sol: puzzle.sol[i],
				q: puzzle.q[i]
			});
		}
		return arr;
	}

	// Code for PIN entry steps
	let cur_step = 0;
	let is_entry_phase = false;
	let cur_puzzle = make_puzzle();
	$: zipped_puzzle = puzzle_zip(cur_puzzle);
	let pin = '';
	let show_pin = false;

	// Code for UI
	let digits = '';
	let started = false;
	let color_odd = "#ffffff";
	let color_even = "cyan";

	function padNum(num: number) {
		return num.toString().padStart(2, '0');
	}

	function rotate(arr: Uint8Array) {
		const elem = arr[0];
		for (var i = 0; i < 9; ++i) {
			arr[i] = arr[i + 1];
		}
		arr[9] = elem;
	}

	async function nextPhase() {
		for (var i = 0; i < 10; ++i) {
			await new Promise((resolve) => setTimeout(resolve, 200));
			rotate(cur_puzzle.sol);
			rotate(cur_puzzle.q);
			[color_odd, color_even] = [color_even, color_odd];
			cur_puzzle = cur_puzzle;
		}
		// Commenting the below line prevents automatic transitioning
		is_entry_phase = true;
	}

	function enterNum(num: number) {
		if (digits.length === 2) {
			return;
		}
		digits += num;
	}
	function delChar() {
		digits = digits.substr(0, digits.length - 1);
	}
	function nextStep() {
		if (digits.length != 2) {
			return;
		}
		const idDigit = Math.floor(Number(digits) / 10);
		const shDigit = Number(digits) % 10;
		let shift_val: number = 0;
		for (const num of cur_puzzle.sol) {
			if (Math.floor(num / 10) == idDigit) {
				shift_val = num % 10;
				break;
			}
		}
		const acDigit = sub_mod10(shDigit, shift_val);
		pin += String(acDigit);
		++cur_step;
		cur_puzzle = make_puzzle();
		digits = '';
		is_entry_phase = false;
	}
</script>

<svelte:head>
	<title>Merkle PIN Entry</title>
	<meta name="description" content="A cognitive trapdoor game based on Merkle puzzles" />
	<link rel="preconnect" href={`http://${import.meta.env.VITE_LOCAL_IP}:8000/`}>
</svelte:head>

<h1>Merkle PIN Entry</h1>

{#if redirect}
	<!-- <video src={url} autoplay></video> -->
	<h2>Redirected!</h2>
{:else if !started}
	<button on:click={() => started=true}>Start Game</button>
{:else if cur_step == 4}
	{#if show_pin}
		<div>Entered PIN: {pin}</div>
	{/if}
	<br />
	<button on:click={() => show_pin = !show_pin}>Toggle PIN Visibility</button>
{:else if is_entry_phase == false}
	<!-- modify next two lines to make clicking phase transition -->
	<!-- on:click={() => (is_entry_phase = true)} -->
	<div class="puzzle" style="--color-odd: {color_odd}; --color-even: {color_even}">
		{#each zipped_puzzle as entry}
			<div class="entry">
				{`${padNum(sub_mod100(entry.sol, entry.q))} + ${padNum(entry.q)} = ?`}
			</div>
		{/each}
	</div>
	{#await nextPhase() catch error}
		<h2>{error}</h2>
	{/await}
{:else if is_entry_phase === true}
	<input
		bind:value={digits}
		name="digits"
		placeholder="Enter Identifier digit followed by Shifted digit"
		type="number"
		readonly
	/>
	<br />
	<div class="keypad">
		<button class="key digit" on:click={() => enterNum(1)}>1</button>
		<button class="key digit" on:click={() => enterNum(2)}>2</button>
		<button class="key digit" on:click={() => enterNum(3)}>3</button>
		<button class="key digit" on:click={() => enterNum(4)}>4</button>
		<button class="key digit" on:click={() => enterNum(5)}>5</button>
		<button class="key digit" on:click={() => enterNum(6)}>6</button>
		<button class="key digit" on:click={() => enterNum(7)}>7</button>
		<button class="key digit" on:click={() => enterNum(8)}>8</button>
		<button class="key digit" on:click={() => enterNum(9)}>9</button>
		<button class="key" on:click={delChar}>Backspace</button>
		<button class="key digit" on:click={() => enterNum(0)}>0</button>
		<button class="key" on:click={nextStep}>Enter</button>
	</div>
{:else}
	<h2>Error: Reached invalid state!</h2>
{/if}

<style>
	input {
		font-size: 1em;
	}
	.keypad {
		display: grid;
		grid-template-columns: repeat(3, 1fr);
		grid-auto-rows: minmax(100px, auto);
		border-style: solid;
	}
	.key {
		display: flex;
		border-style: solid;
		background: white;
		align-items: center;
		justify-content: center;
	}
	.digit {
		font-size: 2em;
	}
	.puzzle {
		display: flex;
		flex-direction: column;
		border-style: solid;
	}
	.entry {
		display: flex;
		border-style: solid;
		align-items: center;
		justify-content: center;
		font-size: 2em;
	}

	.entry:nth-child(odd) {
	    background: var(--color-odd);
	}

	.entry:nth-child(even) {
	    background: var(--color-even);
	}
</style>
