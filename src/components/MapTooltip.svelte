<script>
    export let route_short_name;
    export let route_long_name;
    export let cost_per_rider;
    // export let routeColor;
</script>

<div class="popup-content">
    <h3 class="tooltip-head">Bus: {route_short_name}</h3>
    <p class="tooltip-subhead">{route_long_name}</p>
    <p class="tooltip-text">Cost per rider: {cost_per_rider}</p>
</div>

<style>
    /* :global(#app .maplibregl-popup) {
        max-width: 275px;
    } */
    :global(.maplibregl-popup-content) {
        padding: 0;
    }
    .popup-content {
        font-family: 'BentonSansCond-Regular', sans-serif !important;
    }

    :global(#app .tooltip-head) {
        background-color: var(--grey01);
        color: #FFF;
        font-size: 1.2rem;
        margin-bottom: 7px;
        padding: 8px 8px;
    }

    :global(#app .tooltip-subhead) {
        color: var(--grey-04);
        font-family: 'BentonSansCond-RegItalic', sans-serif;
        font-size: 1rem;
        margin-bottom: 3px;
        padding: 0 8px;
    }

    .tooltip-text {
        /* color: var(--black); */
        font-size: 1rem;
        padding: 0 4px 8px 4px;
    }
</style>