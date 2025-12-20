<script>
    export let stationName;
    export let visits;
</script>

<div class="popup-content">
    <h3 class="station-name">{stationName}</h3>
    <p class="visits">Departures from this station: {visits}</p>
</div>

<style>
    .popup-content {
        font-family: 'BentonSansCond-Reg', sans-serif;
        padding: 4px 2px;
    }

    .station-name {
        color: var(--black);
        font-size: 1.2rem;
        margin-bottom: 7px;
    }

    .visits {
        color: var(--black);
        font-size: 1rem;
    }
</style>