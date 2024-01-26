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
</script>


<h1>SSE Test</h1>

{#if redirect}
	<p>Redirect!</p>
{/if}
