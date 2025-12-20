<script>
    import { onMount, onDestroy } from 'svelte';
    import Maplibregl from 'maplibre-gl';
    import 'maplibre-gl/dist/maplibre-gl.css';
    import MapTooltip from './MapTooltip.svelte';
    import Helper from '../lib/helper-functions.js';
    
    export let data = [];
    export let apiKey;
    
    
    
    let map;
    let mapContainer;
    let animationFrame;
    let currentIndex = 0;
    let isPlaying = false;
    let stationVisitCounts = {};
    
    const stationSize = 4
    const currentStationSize = 6;
    const lineColour = '#AFBEDB';
    const stationColour = '#0062A3';
    const currentStationColour = '#9B3F86';
    const STATION_SIZE_INCREMENT = 0.25;
    const TIME_BETWEEN_STATIONS = 35;  // in milliseconds
    const MAP_STYLE = `https://api.maptiler.com/maps/dataviz/style.json?key=${apiKey}`;

    $: console.log(data)

    // $: {
    //     // Debug: Find entries with invalid coordinates
    //     if (data.length > 0) {
    //         const invalidEntries = data.filter((d, index) => {
    //             const lon = Number(d.lon);
    //             const lat = Number(d.lat);
    //             const isInvalid = isNaN(lon) || isNaN(lat);
    //             if (isInvalid) {
    //                 console.log(`Invalid entry at index ${index}:`, d);
    //             }
    //             return isInvalid;
    //         });
            
    //         if (invalidEntries.length > 0) {
    //             console.log(`Found ${invalidEntries.length} invalid entries:`, invalidEntries);
    //         }
    //     }
    // }
    // Calculate map center from stations
    $: mapCenter = data.length > 0 
        ? [
            data.reduce((sum, d) => sum + Number(d.lon), 0) / data.length,
            data.reduce((sum, d) => sum + Number(d.lat), 0) / data.length
        ]
        : [-123.12, 49.27];

    onMount(() => {
        map = new Maplibregl.Map({
            container: mapContainer,
            style: MAP_STYLE,
            center: mapCenter,
            zoom: 12,
            pitch: 45,
            bearing: 0
        });


        map.on('load', () => {
            // Add source for trail line
            map.addSource('trail-line', {
            type: 'geojson',
            data: {
                type: 'Feature',
                properties: {},
                geometry: {
                    type: 'LineString',
                    coordinates: []
                }
            }
        });

        // Add line layer for the trail
        map.addLayer({
            id: 'trail-line-layer',
            type: 'line',
            source: 'trail-line',
            layout: {
                'line-join': 'round',
                'line-cap': 'round'
            },
            paint: {
                'line-color': lineColour,
                'line-width': 4,
                'line-opacity': 0.8
            }
        });

        // Add source for visited stations (persistent dots)
        map.addSource('visited-stations', {
            type: 'geojson',
            data: {
                type: 'FeatureCollection',
                features: []
            }
        });

        // Add layer for visited stations with dynamic sizing based on visit count
        map.addLayer({
            id: 'visited-stations-layer',
            type: 'circle',
            source: 'visited-stations',
            paint: {
                // 'circle-radius': stationSize,
                'circle-radius': ['get', 'size'],
                'circle-color': stationColour,
                'circle-stroke-width': 1,
                'circle-stroke-color': '#ffffff',
                'circle-opacity': 0.9
            }
        });

        // Add source for current position (moving dot)
        map.addSource('current-position', {
            type: 'geojson',
            data: {
                type: 'Feature',
                properties: {},
                geometry: {
                    type: 'Point',
                    coordinates: [data[0].lon, data[0].lat]
                }
            }
        });

        // Add layer for current position
        map.addLayer({
            id: 'current-position-layer',
            type: 'circle',
            source: 'current-position',
            paint: {
                'circle-radius': currentStationSize,
                'circle-color': currentStationColour,
                'circle-stroke-width': 1,
                // 'circle-stroke-color': '#ffffff',
                'circle-opacity': 1
            }
        });

        // Add labels for stations
        // map.addLayer({
        //     id: 'station-labels',
        //     type: 'symbol',
        //     source: 'visited-stations',
        //     layout: {
        //         'text-field': ['get', 'station_name'],
        //         'text-size': 12,
        //         'text-offset': [0, 1.5],
        //         'text-anchor': 'top'
        //     },
        //     paint: {
        //         'text-color': '#1f2937',
        //         'text-halo-color': '#ffffff',
        //         'text-halo-width': 2
        //     }
        // });

        // Initialize with first station
        updateMap(0);

        // Autoplay animation once map is loaded
        isPlaying = true;
        animate();

        // Add click event for station popups
        map.on('click', 'visited-stations-layer', e => {
            const coordinates = e.features[0].geometry.coordinates.slice();
            const { station_name, visits } = e.features[0].properties;

            // Ensure that if the map is zoomed out such that multiple
            // copies of the feature are visible, the popup appears
            // over the copy being pointed to.
            while (Math.abs(e.lngLat.lng - coordinates[0]) > 180) {
                coordinates[0] += e.lngLat.lng > coordinates[0] ? 360 : -360;
            }

            // Create a container for the Svelte component
            const popupContainer = document.createElement('div');

            // Mount the tooltip
            new MapTooltip({
                target: popupContainer,
                props: {
                    stationName: station_name,
                    visits: visits
                }
            });

            new Maplibregl.Popup()
                .setLngLat(coordinates)
                .setDOMContent(popupContainer)
                .addTo(map);
        });

        // Change cursor to a pointer when hovering
        map.on('mouseenter', 'visited-station-layers', () => {
            map.getCanvas().style.cursor = 'pointer';
        });

        // Change it back to default when leaving
        map.on('mouseleave', 'visited-stations-layer', () => {
            map.getCanvas().style.cursor = '';
        });
    });
    });

    onDestroy(() => {
        if (animationFrame) {
            clearTimeout(animationFrame);
        }
        if (map) {
            map.remove();
        }
    });

    function updateMap(index) {
        if (!map || index >= data.length) return;

        const currentStation = data[index];
        const stationId = currentStation.id || currentStation.station_name;

        // Update visit count for current station
        if (!stationVisitCounts[stationId]) {
            stationVisitCounts[stationId] = 0;
        }
        stationVisitCounts[stationId]++;

        // Get all unique stations up to current index
        // This is so we only draw each station once
        const visitedStationsMap = new Map();
        
        for (let i = 0; i <= index; i++) {
            const station = data[i];
            const id = station.id || station.station_name;
            visitedStationsMap.set(id, station);
        }

        const visitedStations = Array.from(visitedStationsMap.values());
    
        // Update visited stations (persistent dots) with dynamic sizes based on visit count
        map.getSource('visited-stations').setData({
            type: 'FeatureCollection',
            features: visitedStations.map(data => {
                const id = data.id || data.station_name;
                const visitCount = stationVisitCounts[id] || 1;
                const size = stationSize + (visitCount - 1) * STATION_SIZE_INCREMENT;

                return {
                    type: 'Feature',
                    properties: {
                        station_name: data.station_name,
                        id: id,
                        size: size,
                        visits: visitCount
                    },
                    geometry: {
                        type: 'Point',
                        coordinates: [data.lon, data.lat]
                    }
                };
            })
            // features: visitedStations.map(data => ({
            //     type: 'Feature',
            //     properties: {
            //         name: data.station_name,
            //         id: data.id
            //     },
            //     geometry: {
            //         type: 'Point',
            //         coordinates: [data.lon, data.lat]
            //     }
            // }))
        });

        // Update trail line
        const lineCoordinates = index > 0 
            ? [
                [data[index - 1].lon, data[index - 1].lat],
                [data[index].lon, data[index].lat]
            ]
            : [];

        map.getSource('trail-line').setData({
            type: 'Feature',
            properties: {},
            geometry: {
                type: 'LineString',
                coordinates: lineCoordinates
            }
        });

        // Update current position
        const current = data[index];
        map.getSource('current-position').setData({
            type: 'Feature',
            properties: {
                name: current.name
            },
            geometry: {
                type: 'Point',
                coordinates: [current.lon, current.lat]
            }
        });
    }

    function animate() {
        if (currentIndex < data.length - 1) {
            currentIndex += 1;
            updateMap(currentIndex);
            animationFrame = setTimeout(() => {
                if (isPlaying) {
                animate();
                }
            }, TIME_BETWEEN_STATIONS);
        } else {
            isPlaying = false;
        }
    }

    function togglePlay() {
        isPlaying = !isPlaying;
        
        if (isPlaying) {
            animate();
        } else if (animationFrame) {
            clearTimeout(animationFrame);
        }
    }

    // function pausePlay() {
    //     isPlaying = !isPlaying;
    // }

    function reset() {
        isPlaying = false;
        currentIndex = 0;
        
        if (animationFrame) {
            clearTimeout(animationFrame);
        }
        if (map && map.getSource('visited-stations')) {
            updateMap(0);
        }

        // restart playback
        isPlaying = true;
        animate();        
    }
</script>

<div class="container">
    <div bind:this={mapContainer} class="map"></div>
    
    <div class="display">
        <h2 class="info">{data[currentIndex].depart_string}</h2>
        <p class="info">Trips: {Helper.numberWithCommas(currentIndex)}</p>
    </div>
    <div class="controls">
        <button on:click={togglePlay} class="btn">
            {#if isPlaying}
                ⏸ Pause
            {:else}
                ▶ Play
            {/if}
        </button>
        <button on:click={reset} class="btn">
            ↻ Restart
        </button>
        <!-- <button on:click={pausePlay} class="btn">
            ⏸ Pause
        </button> -->
    </div>
</div>

<style>
    .container {
        font-family: 'BentonSansCond-Reg', sans-serif;
        height: 100vh;
        max-height: 500px;
        position: relative;
        width: 100%;
    }

    .map {
        height: 100%;
        max-height: 500px;
        width: 100%;
    }

    .display {
        border-radius: 8px;
        top: 10px;
        left: 10px;
        position: absolute;
    }
    #app .display h2 {
        color: var(--blue01);
        font-size: 2rem;
        margin-bottom: 7px;
    }

    #app .display p {
        color: var(--blue03);
        font-family: 'BentonSansCond-Bold', sans-serif;
        font-size: 1.2rem;
    }

    .controls {
        position: absolute;
        top: 80px;
        left: 10px;
        display: flex;
        gap: 10px;
        align-items: center;
        z-index: 1;
    }

    .btn {
        padding: 4px 8px;
        border: none;
        border-radius: 6px;
        background: #6D8EBF;
        color: #FFF;
        cursor: pointer;
        font-size: 0.9rem;
        font-weight: 500;
        transition: background 0.2s;
        }

        .btn:hover {
            background: #0062A3;
        }

        .info {
            /* font-size: 14px; */
            color: #231F20;
            font-weight: 500;
        }
</style>