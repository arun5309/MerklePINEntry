<script lang="ts">
	var redirect = false;
	import banner from '$lib/images/Banners_Mathematical_Merkle.png';
	// var url = '';
	// const evtSource = new EventSource(`http://${import.meta.env.VITE_LOCAL_IP}:8000/stream`);
	// evtSource.onmessage = function (event) {
	// 	if (event.data) {
	// 		redirect = true;
	// 		window.location.replace(event.data);
	// 		// window.location.href = event.data;
	// 		// url = event.data; // Frame URL
	// 		// url = `http://${import.meta.env.VITE_LOCAL_IP}:8000/${event.data}`; // Video URL
	// 	} else {
	// 		redirect = false;
	// 	}
	// };

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
	let color_odd = '#ffffff';
	let color_even = 'cyan';

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
		for (var i = 0; i < 20; ++i) {
			await new Promise((resolve) => setTimeout(resolve, 650));
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
			if (Math.floor(num / 10) === idDigit) {
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
		if (pin.length === 4) {
			finish_transition();
		}
	}

	type GetPointsResponse = {
		uid: number | null;
		points: number | null;
		actual_pin: string;
	};

	type CreateInstanceResponse = {
		iid: number;
	};

	type UpdateInstanceResponse = {
		iid: number;
	};

	let uid: string = '';
	let uid_valid: boolean = false;
	$: uid_valid = check_uid_valid(uid);
	let iid: number = -1;
	let actual_pin: string = '';
	const USER_ID_LENGTH = 3;

	function reset() {
		cur_step = 0;
		is_entry_phase = false;
		cur_puzzle = make_puzzle();
		pin = '';
		show_pin = false;
		// Code for UI
		digits = '';
		color_odd = '#ffffff';
		color_even = 'cyan';

		started = false;
		show_pin = false;
		uid_valid = false;
		iid = -1;
		actual_pin = '';
		const temp = uid;
		uid = '';
		uid = temp;
	}

	async function get_points(): Promise<GetPointsResponse> {
		const url = `https://142.93.219.243.nip.io/points/${uid.toLowerCase()}`;
		const request = new Request(url, { method: 'GET' });
		const data = await fetch(request);
		return <GetPointsResponse>(<unknown>data.json());
	}

	function normalize() {
		uid = uid;
	}

	function set_actual_pin(val: string) {
		actual_pin = val;
	}

	function isAlphaNumeric(str: string): boolean {
		let code, i, len;
		for (i = 0, len = str.length; i < len; i++) {
			code = str.charCodeAt(i);
			if (!(code > 47 && code < 58) && !(code > 64 && code < 91) && !(code > 96 && code < 123)) {
				return false;
			}
		}
		return true;
	}

	function check_uid_valid(uid_cand: string): boolean {
		return uid_cand.length === USER_ID_LENGTH && isAlphaNumeric(uid_cand);
	}

	function progress_transition() {
		if (!uid_valid) return;
		const url = 'https://142.93.219.243.nip.io/create_instance';
		const data = {
			game_id: 'NM',
			user_id: uid.toLowerCase()
		};
		const request = new Request(url, {
			method: 'POST',
			body: JSON.stringify(data),
			headers: new Headers({
				'Content-Type': 'application/json; charset=UTF-8'
			})
		});
		fetch(request).then((create_instance_value_temp) => {
			create_instance_value_temp.json().then((temp) => {
				const create_instance_value = <CreateInstanceResponse>(<unknown>temp);
				iid = create_instance_value.iid;
				started = true;
			});
		});
	}

	function finish_transition() {
		const url = 'https://142.93.219.243.nip.io/update_instance';
		const data = {
			iid_value: iid,
			result_pin: pin
		};
		const request = new Request(url, {
			method: 'POST',
			body: JSON.stringify(data),
			headers: new Headers({
				'Content-Type': 'application/json; charset=UTF-8'
			})
		});
		fetch(request).then((instance_response_value_temp) => {
			instance_response_value_temp.json().then((temp) => {
				const instance_response_value = <UpdateInstanceResponse>(<unknown>temp);
				if (iid !== instance_response_value.iid) {
					alert('Reached invalid state, please report bug!');
				}
				// game_state = GameState.FINISH;
			});
		});
	}
	function gameHeader(num: number) {
		if (num === 1) {
			return '1st';
		} else if (num === 2) {
			return '2nd';
		} else if (num === 3) {
			return '3rd';
		} else if (num == 4) {
			return '4th';
		} else {
			console.log('some number>4 passed into gameHeader function');
		}
	}
</script>

<svelte:head>
	<title>Merkle PIN Entry</title>
	<meta name="description" content="A cognitive trapdoor game based on Merkle puzzles" />
	<link rel="preconnect" href={`http://${import.meta.env.VITE_LOCAL_IP}:8000/`} />
</svelte:head>

<!-- <h1>Merkle PIN Entry</h1> -->

{#if redirect}
	<!-- <video src={url} autoplay></video> -->
	<h2>Redirected!</h2>
{:else if !started}
	<img src={banner} alt="Mathematical Merkle" />
	<br />
	<input
		type="text"
		placeholder="User ID"
		bind:value={uid}
		on:change={normalize}
		maxlength={USER_ID_LENGTH}
		name="userid"
		id="userid"
	/>
	{#if uid_valid}
		{#await get_points()}
			<p>Validating User ID...</p>
		{:then get_points_value}
			{#if get_points_value.uid !== null}
				{(set_actual_pin(get_points_value.actual_pin), '')}
				<p style="color: green">Points: {get_points_value.points}</p>
			{:else}
				<p style="color: red">Invalid User ID!</p>
			{/if}
		{:catch error}
			{(console.log(error), '')}
			<p style="color: purple">Network Error: Unable to check validity of User ID!</p>
		{/await}
	{/if}
	<br />
	<button on:click={progress_transition}>Start Game</button>
{:else if cur_step === 4}
	<img src={banner} alt="Mathematical Merkle" />
	<br />
	{#if show_pin}
		<div>Entered PIN: {pin}</div>
	{/if}
	<br />
	{#if actual_pin === pin}
		<p style="color: green">Congratulations on entering the correct PIN!</p>
	{:else}
		<p style="color: red">Incorrect PIN entered, no points earned!</p>
	{/if}
	<br />
	{#await get_points()}
		<p>Fetching points...</p>
	{:then get_points_value}
		{#if get_points_value.uid !== null}
			<p style="color: green">Points: {get_points_value.points}</p>
		{:else}
			<p style="color: red">Invalid User ID!</p>
		{/if}
	{:catch error}
		{(console.log(error), '')}
		<p style="color: purple">Network Error: Unable to fetch points!</p>
	{/await}
	<br />
	<button on:click={() => (show_pin = !show_pin)}>Toggle Visibility of Entered PIN</button>
	<br />
	<button on:click={reset}>Play Again</button>
	<br />
	<button><a href="https://142.93.219.243.nip.io/">Checkout Other Games</a></button>
{:else if is_entry_phase === false}
	<h1><strong>{gameHeader(pin.length + 1)} digit</strong> | Round 1</h1>
	<!-- modify next two lines to make clicking phase transition -->
	<!-- on:click={() => (is_entry_phase = true)} -->
	<div class="puzzle" style="--color-odd: {color_odd}; --color-even: {color_even}">
		{#each zipped_puzzle as entry}
			<div class="entry">
				{`${padNum(sub_mod100(entry.sol, entry.q))} + ${padNum(entry.q)} = ?`}
			</div>
		{/each}
	</div>
	<h2 style="text-align:center;font-size:1.5em;">
		Each equation is marching upwards. Track one and just remember it.
	</h2>
	{#await nextPhase() catch error}
		<h2>{error}</h2>
	{/await}
{:else if is_entry_phase === true}
	<h1><strong>{gameHeader(pin.length + 1)} digit</strong> | Round 2</h1>
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
	<h2 style="text-align:center;">
		Now solve the problem and consider only the last 2 digits. After this, input the first digit <strong
			>of the solution</strong
		>
		normally. For the second digit of the solution, take it and add your current PIN digit then
		<strong>input only the last digit</strong>. Take your time.
	</h2>
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
