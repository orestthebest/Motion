<script>

import { onMount } from 'svelte';

let alpha = $state(null);
let beta = $state(null);
let gamma = $state(null);

 onMount(() => {
    window.addEventListener('deviceorientation', (event)=>{
        alpha = event.alpha;
        beta = event.beta;
        gamma = event.gamma;

     });
});
</script>


<h1>Device Orientation</h1>


<p>Alpha: {alpha}</p>
<p>Beta: {beta}</p>
<p>Gamma: {gamma}</p>