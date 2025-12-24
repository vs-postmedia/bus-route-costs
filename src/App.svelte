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
    <h1>Average cost per passenger to operate TransLink bus routes.</h1>
    <p class="subhead"></p>
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
    <p class="note">NOTE: Cost per passenger was calculated by dividing the hourly cost to operate a bus by the average number of passengers per hour in 2024. TransLink provided hourly costs for shuttle buses and an average hourly cost for all other bus types (standard, articulated, etc.).</p>
    <p class="source">Source:  <a href="https://www.translink.ca/plans-and-projects/strategies-plans-and-guidelines/managing-the-transit-network#2024-transit-service-performance-review" target="_blank">TransLink</a>, Postmedia analysis</p>
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
		line-height: 2.1rem;
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
