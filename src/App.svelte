<script>
    // COMPONENTS
    import { onMount } from 'svelte';
    import { csvParse } from 'd3-dsv';
    import Map from "$components/Map.svelte";

    

    // DATA
    // import data from "$data/data.js";
    import busRoutes from '$data/bus-routes.js';
    const mapDataUrl = 'https://vs-postmedia-data.sfo2.digitaloceanspaces.com/misc/mobi-top-bike-data.csv';

    // VARIABLES
    let mapData;
    
    // create .env in root dir & add VITE_MAPTILER_API_KEY for Map.svelte
    const apiKey = import.meta.env.VITE_MAPTILER_API_KEY;

    async function fetchData(url) {
        const resp = await fetch(url);
        const data = await resp.text();

        return csvParse(data);
    }


    async function init() {
        // fetch map data
        mapData = await fetchData(mapDataUrl);

        // console.log(mapData)
    }

    onMount(init);
</script>

<header>
    <h1>VS SvelteKit Template</h1>
    <p class="subhead">Visit <a href="https://kit.svelte.dev">kit.svelte.dev</a> to read the documentation</p>
</header>

<main>
    {#if mapData}
        <Map
            apiKey={apiKey}
            data={mapData}
            lineData={busRoutes[0]}
        />
    {/if}
</main>

<footer>
    <p class="note">NOTE: tk.</p>
    <p class="source">Source:  <a href="https:vancouversun.com" target="_blank">TK</a></p>
</footer>
  
<style>
    @import '$css/normalize.css';
    @import '$css/fonts.css';
    @import '$css/colors.css';
    @import '$css/app.css';

    header {
		margin-bottom: 2rem;
	}
	header > h1 {
		text-align: center;
	}
	header .subhead {
		margin: 0 auto;
		max-width: 525px;
		text-align: center;
	}

    /* COMBOBOX SELECTOR */
  	:global(.svelte-select) {
		margin: 1rem auto !important;
		max-width: 250px;
  	}
  	:global(input:focus) {
		outline: none;
  	}

	:global(
		.svelte-select .selected-item,
		.svelte-select .item,
		.svelte-select input
	) {
		font-family: 'BentonSansCond-Regular', sans;
	}
</style>
