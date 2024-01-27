<script>    	
    var redirect = false;
    const evtSource = new EventSource(`http://${import.meta.env.VITE_LOCAL_IP}:8000/stream`);
    evtSource.onmessage = function(event) {
    	if(event.data == "rick roll") {
    	    redirect = true;
    	    // window.location.replace(import.meta.env.VITE_RR);
    	    window.location.href = import.meta.env.VITE_RR;
    	}
    	else {
    	    redirect = false;
    	}
    };

    function rand_digit() {
        for(;;) {
            const array = new Uint8Array(1);
            self.crypto.getRandomValues(array);
            if(array[0] < 250) {
                return array[0] % 10;
            }
        }
    }

    function rand_digit_min_i(i) {
        for(;;) {
            const j = rand_digit();
            if(j >= i) {
                return j;
            }
        }
    }

    function rand_perm() {
        let array = new Uint8Array(10);
        for(var i=0;i<10;++i) {
            array[i] = i;
        }
        for(var i=0;i<9;++i) {
            const j = rand_digit_min_i(i);
            const temp = array[i];
            array[i] = array[j];
            array[j] = temp;
        }    
        return array;
    }

    function sub_mod100(a, b) {
        return (((a - b) % 100) + 100) % 100;
    }

    function sub_mod10(a, b) {
        return (((a - b) % 10) + 10) % 10;
    }

    function make_puzzle() {
        let puzzle_sol = rand_perm();
        let puzzle_q = new Uint8Array(10);
        for(var i=0;i<10;++i) {
            puzzle_sol[i] = Number(String(puzzle_sol[i]) + String(rand_digit()));
            puzzle_q[i] = Number(String(rand_digit()) + String(rand_digit()));
        }
        return {
          "sol": puzzle_sol,
          "q": puzzle_q  
        };
    }

    function puzzle_zip(puzzle) {
        let arr = [];
        for(var i=0;i<10;++i) {
            arr.push({
                "sol": puzzle.sol[i],
                "q": puzzle.q[i]
            });
        }
        return arr;
    }

    // Code for PIN entry steps
    let cur_step = 0;
    let is_entry_phase = false;
    let cur_puzzle = make_puzzle();
    $: zipped_puzzle = puzzle_zip(cur_puzzle);
    let pin = "";
    let show_pin = false;

    // Code for UI
    let digits = "";
    let unique = {};

    function padNum(num) {
        return num.toString().padStart(2, "0");
    }

    function rotate(arr) {
        const elem = arr[0];
        for(var i=0; i < 9; ++i) {
            arr[i] = arr[i + 1];
        }
        arr[9] = elem;
    }

    async function nextPhase() {
        for(var i = 0; i < 10; ++i) {
            await new Promise(resolve => setTimeout(resolve, 200));
            rotate(cur_puzzle.sol);
            rotate(cur_puzzle.q);
            cur_puzzle = cur_puzzle;
        }
        // Commenting the below line prevents automatic transitioning
        is_entry_phase = true;
    }

    function unique_transition() {
        unique = {};
        return true;
    }

    function enterNum(num) {
        if(digits.length === 2) {
            return;
        }
        digits += num;
    }
    function delChar() {
        digits = digits.substr(0, digits.length-1);
    }
    function nextStep() {
        if(digits.length != 2) {
            return;
        }
        const idDigit = Math.floor(digits / 10);
        const shDigit = digits % 10;
        let shift_val;
        for(const num of cur_puzzle.sol) {
            if(Math.floor(num / 10) == idDigit) {
                shift_val = num % 10;
                break;
            }
        }
        const acDigit = sub_mod10(shDigit, shift_val);
        pin += String(acDigit);
        ++cur_step;
        cur_puzzle = make_puzzle();
        digits = "";
        is_entry_phase = false;
    }
</script>

<svelte:head>
	<title>Merkle PIN Entry</title>
	<meta name="description" content="A cognitive trapdoor game based on Merkle puzzles" />
</svelte:head>


<h1>Merkle PIN Entry</h1>

{#if redirect}
	<p>Redirect!</p>
{:else if cur_step == 4}
    {#if show_pin}
        <div>Entered PIN: {pin}</div>
    {/if}
    <br/>
    <button on:click={() => show_pin = !show_pin}>Toggle PIN Visibility</button>
{:else if is_entry_phase == false && unique_transition()}
    <!-- modify next line to prevent clicks from phase transitioning -->
    <div class="puzzle" on:click={() => is_entry_phase=true}>
        {#each zipped_puzzle as entry}
            {#key unique}
                <div class="entry">
                    {`${padNum(sub_mod100(entry.sol, entry.q))} + ${padNum(entry.q)} = ?`}
                </div>
            {/key}
        {/each}
    </div>
    {#await nextPhase() catch error}
        <h2>{error}</h2>
    {/await}
{:else if is_entry_phase === true}
    <input bind:value={digits} name="digits" placeholder="Enter Identifier digit followed by Shifted digit" type="number" readonly />
    <br/>
    <div class="keypad">
        <div class="key digit" on:click={() => enterNum(1)}>1</div>
        <div class="key digit" on:click={() => enterNum(2)}>2</div>
        <div class="key digit" on:click={() => enterNum(3)}>3</div>
        <div class="key digit" on:click={() => enterNum(4)}>4</div>
        <div class="key digit" on:click={() => enterNum(5)}>5</div>
        <div class="key digit" on:click={() => enterNum(6)}>6</div>
        <div class="key digit" on:click={() => enterNum(7)}>7</div>
        <div class="key digit" on:click={() => enterNum(8)}>8</div>
        <div class="key digit" on:click={() => enterNum(9)}>9</div>
        <div class="key" on:click={delChar}>Backspace</div>
        <div class="key digit" on:click={() => enterNum(0)}>0</div>
        <div class="key" on:click={nextStep}>Enter</div>
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
        background: white;
        align-items: center;
        justify-content: center;
        font-size: 2em;
    }
</style>
