<script>
    import { onMount, onDestroy } from 'svelte';
    import Maplibregl from 'maplibre-gl';
    import 'maplibre-gl/dist/maplibre-gl.css';
    import MapTooltip from './MapTooltip.svelte';
    import Helper from '../lib/helper-functions.js';
    
    export let data = [];
    export let lineData;
    export let apiKey;
    
    
    console.log(lineData)
    
    let map;
    let mapContainer;
    let costBuckets = [];

    const mapZoom = 9;
    const lineColour = '#0062A3';
    const MAP_STYLE = `https://api.maptiler.com/maps/dataviz/style.json?key=${apiKey}`;

    // Color scale for 5 buckets (low to high cost)
    const colorScale = [
        '#0062A3', // Dark blue - lowest cost
        '#829DC7', // Light blue
        '#F6B31C', // Yellow
        '#F58745', // Orange
        '#E35D42'  // Red - highest cost
    ];

    // Calculate cost buckets from lineData
    $: if (lineData && lineData.features && lineData.features.length > 0) {
            const costs = lineData.features.map(f => Number(f.properties.cost_per_rider));
            const minCost = Math.min(...costs);
            const maxCost = Math.max(...costs);
            const bucketSize = (maxCost - minCost) / 5;
            
            costBuckets = [
                minCost + bucketSize,
                minCost + bucketSize * 2,
                minCost + bucketSize * 3,
                minCost + bucketSize * 4
            ];

            // Update the layer style if map already exists
            if (map && map.getLayer('bus-routes-layer')) {
                map.setPaintProperty('bus-routes-layer', 'line-color', [
                    'step',
                    ['to-number', ['get', 'cost_per_rider']],
                    colorScale[0],
                    costBuckets[0], colorScale[1],
                    costBuckets[1], colorScale[2],
                    costBuckets[2], colorScale[3],
                    costBuckets[3], colorScale[4]
                ]);
            }
        }



    // Calculate map center from stations
    // $: mapCenter = data.length > 0 
    //     ? [
    //         data.reduce((sum, d) => sum + Number(d.lon), 0) / data.length,
    //         data.reduce((sum, d) => sum + Number(d.lat), 0) / data.length
    //     ]
    //     : [-123.12, 49.27];
    
    // calculate map centre from bbox of lines
    $: mapCenter = lineData && lineData.features && lineData.features.length > 0
    ? (() => {
        // Get all coordinates from all linestrings
        const allCoords = lineData.features.flatMap(feature => 
            feature.geometry.coordinates
        ).flat();
        
        // Calculate bounding box
        const lons = allCoords.map(coord => coord[0]);
        const lats = allCoords.map(coord => coord[1]);
        
        const minLon = Math.min(...lons);
        const maxLon = Math.max(...lons);
        const minLat = Math.min(...lats);
        const maxLat = Math.max(...lats);
        
        // Return center of bbox
        return [
            (minLon + maxLon) / 2,
            (minLat + maxLat) / 2
        ];
    })()
    : [-123.12, 49.27];

    onMount(() => {
        let popup = null;

        map = new Maplibregl.Map({
            container: mapContainer,
            style: MAP_STYLE,
            center: mapCenter,
            zoom: mapZoom,
            pitch: 30,
            bearing: 0
        });


        map.on('load', () => {
            // Add source for trail line
            map.addSource('bus-routes', {
            type: 'geojson',
            data: lineData
        });

        // Add line layer for the trail
        map.addLayer({
            id: 'bus-routes-layer',
            type: 'line',
            source: 'bus-routes',
            layout: {
                'line-join': 'round',
                'line-cap': 'round'
            },
            paint: {
                // 'line-color': lineColour,
                'line-color': [
                    'step',
                    ['to-number', ['get', 'cost_per_rider']],
                    colorScale[0],
                    costBuckets[0], colorScale[1],
                    costBuckets[1], colorScale[2],
                    costBuckets[2], colorScale[3],
                    costBuckets[3], colorScale[4]
                ],
                'line-width': 3,
                'line-opacity': 0.8
            }
        });

        // Function to create and show popup
        const showPopup = (e) => {
            // Change cursor to pointer on hover
            map.getCanvas().style.cursor = 'pointer';

            const coordinates = [e.lngLat.lng, e.lngLat.lat]; // line string
            const { route_short_name, route_long_name, cost_per_rider } = e.features[0].properties;

            // Create a container for the tootlip
            const popupContainer = document.createElement('div');

            // Mount the tooltip
            new MapTooltip({
                target: popupContainer,
                props: {
                    route_short_name: route_short_name,
                    route_long_name: route_long_name,
                    cost_per_rider: `$${cost_per_rider}`
                }
            });

            // Remove existing popup if any
            if (popup) {
                popup.remove();
            }

            popup = new Maplibregl.Popup({ closeButton: false })
                .setLngLat(coordinates)
                .setDOMContent(popupContainer)
                .addTo(map);
        };

        // Add click event for station popups
        map.on('click', 'bus-routes-layer', showPopup);

        // Change cursor to a pointer when hovering
        map.on('mouseenter', 'bus-routes-layer', showPopup);

        // Change it back to default when leaving
        map.on('mouseleave', 'bus-routes-layer', () => {
            // Change cursor back on leave
            map.getCanvas().style.cursor = '';
            
            // if (popup) {
            //     popup.remove();
            //     popup = null;
            // }
        });
    });
    });

    onDestroy(() => {
        if (map) {
            map.remove();
        }
    });
</script>

<div class="container">
    <div bind:this={mapContainer} class="map"></div>
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