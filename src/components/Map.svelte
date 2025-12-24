<script>
    import { onMount, onDestroy } from 'svelte';
    import Maplibregl from 'maplibre-gl';
    import 'maplibre-gl/dist/maplibre-gl.css';
    import MapTooltip from './MapTooltip.svelte';
    import Helper from '../lib/helper-functions.js';
    
    export let data = [];
    export let lineData;
    export let apiKey;
    
    
    let map;
    let popup = null;
    let mapContainer;
    let costBuckets = [];
    let searchQuery = '';
    let filteredRoutes = [];
    let showDropdown = false;
    let selectedRoute = null;

    const mapZoom = 9;
    const LineOpacityDim = 0;
    const LineOpacityFull = 1;
    const searchResultLimit = 5;
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

    
    /*
    ** REACTIVE VARIABLES
    */
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

    // Filter routes based on search query
    $: if (lineData && lineData.features) {
        if (searchQuery.trim()) {
            filteredRoutes = lineData.features.filter(feature => 
                feature.properties.route_short_name
                    .toLowerCase()
                    .includes(searchQuery.toLowerCase())
            )
            // sort numerically
            .sort((a, b) => {
                const numA = parseInt(a.properties.route_short_name);
                const numB = parseInt(b.properties.route_short_name);
                return numA - numB;
            })
             // Limit to XX results
            .slice(0, searchResultLimit);
            
            showDropdown = filteredRoutes.length > 0;
        } else {
            filteredRoutes = [];
            showDropdown = false;
        }
    }

    
    /*
    ** FUNCTIONS
    */
    function getRouteColour(cost_per_rider) {
        const cost = Number(cost_per_rider);
        let routeColor = colorScale[0];

        if (costBuckets.length > 0) {
            if (cost >= costBuckets[3]) {
                routeColor = colorScale[4];
            } else if (cost >= costBuckets[2]) {
                routeColor = colorScale[3];
            } else if (cost >= costBuckets[1]) {
                routeColor = colorScale[2];
            } else if (cost >= costBuckets[0]) {
                routeColor = colorScale[1];
            }
        }
        
        return routeColor;
    }

    function showFixedPopup(popupContainer) {
        // Add fixed popup to container instead of map
        const fixedPopupWrapper = document.createElement('div');
        fixedPopupWrapper.className = 'fixed-popup-container';
        
        const closeButton = document.createElement('button');
        closeButton.className = 'fixed-popup-close';
        closeButton.innerHTML = '×';
        
        closeButton.onclick = () => {
            fixedPopupWrapper.remove();
            selectedRoute = null;
            if (map && map.getLayer('bus-routes-layer')) {
                map.setPaintProperty('bus-routes-layer', 'line-opacity', LineOpacityFull);
            }
        };

        fixedPopupWrapper.appendChild(closeButton);
        fixedPopupWrapper.appendChild(popupContainer);

        document.querySelector('.container').appendChild(fixedPopupWrapper);
    };

    // Function to show popup for a route when searched via searchbar
    function searchRoutes(feature) {
        if (!map || !feature) return;

        // Get the center of the route's coordinates
        const coords = feature.geometry.coordinates;
        const centerIndex = Math.floor(coords.length / 2);
        const centerCoord = coords[centerIndex];
        const coordinates = centerCoord[0]; // [lon, lat]

        const { route_short_name, route_long_name, cost_per_rider } = feature.properties;

        // store selected route
        selectedRoute = route_short_name;
        
        // highlight selected route
        map.setPaintProperty('bus-routes-layer', 'line-opacity', [
            'case',
            ['==', ['get', 'route_short_name'], route_short_name],
            LineOpacityFull,
            LineOpacityDim
        ]);

        // Container for the popup component
        const popupContainer = document.createElement('div');
        popupContainer.className = 'fixed-popup';

        // Mount the tooltip
        new MapTooltip({
            target: popupContainer,
            props: {
                route_short_name: route_short_name,
                route_long_name: route_long_name,
                cost_per_rider: `$${cost_per_rider}`,
                route_color: getRouteColour(cost_per_rider)
            }
        });

        // Remove existing popup if any
        if (popup) {
            popup.remove();
        }

        // show popup
        showFixedPopup(popupContainer);

        // show popup
        // popup = new Maplibregl.Popup({ closeButton: true })
        //     .setLngLat(coordinates)
        //     .setDOMContent(popupContainer)
        //     .addTo(map);

        // // reset opacity on close
        // popup.on('close', () => {
        //     selectedRoute = null;
        //     if (map && map.getLayer('bus-routes-layer')) {
        //         map.setPaintProperty('bus-routes-layer', 'line-opacity', LineOpacityFull);
        //     }
        // });

        // Fly to the route
        // map.flyTo({
        //     center: coordinates,
        //     zoom: 13,
        //     duration: 1000
        // });

        // Clear search
        searchQuery = '';
        showDropdown = false;
    }


    onMount(() => {
        map = new Maplibregl.Map({
            container: mapContainer,
            style: MAP_STYLE,
            center: mapCenter,
            zoom: mapZoom,
            pitch: 30,
            bearing: 0
        });

        map.addControl(new Maplibregl.NavigationControl({
            visualizePitch: true,
            visualizeRoll: true,
            showZoom: true,
            showCompass: true
        }));


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
            // Disble hover if route is selectd in search
            if (selectedRoute) return;

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
                    cost_per_rider: `$${cost_per_rider}`,
                    route_color: getRouteColour(cost_per_rider)
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
            
            if (popup && !selectedRoute) {
                popup.remove();
                popup = null;
            }
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
    <!-- SEARCH BOX -->
    <div class="search-container">
        <input
            type="text"
            placeholder="Lookup a bus route..."
            bind:value={searchQuery}
            class="search-input"
        />
        {#if showDropdown}
            <div class="dropdown">
                {#each filteredRoutes as route}
                    <button 
                        class="dropdown-item"
                        on:click={() => searchRoutes(route)}
                    >
                        <strong>{route.properties.route_short_name}</strong> - {route.properties.route_long_name}
                    </button>
                {/each}
            </div>
        {/if}
    </div>

    <!-- MAP -->
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

    /* SEARCH BAR */
    .search-container {
        position: absolute;
        top: 10px;
        left: 10px;
        z-index: 1;
        width: 175px;
    }

    .search-input {
        width: 100%;
        padding: 10px 12px;
        border: none;
        border-radius: 6px;
        background: white;
        box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
        font-size: 14px;
        font-family: 'BentonSansCond-Reg', sans-serif;
    }

    .search-input:focus {
        outline: 2px solid #0062A3;
    }

    .dropdown {
        background: white;
        border-radius: 6px;
        box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
        margin-top: 4px;
        max-height: 200px;
        overflow-y: auto;
    }

    .dropdown-item {
        width: 100%;
        padding: 10px 12px;
        border: none;
        background: white;
        text-align: left;
        cursor: pointer;
        font-size: 14px;
        font-family: 'BentonSansCond-Reg', sans-serif;
        border-bottom: 1px solid #f0f0f0;
    }

    .dropdown-item:hover {
        background: #f5f5f5;
    }

    .dropdown-item:last-child {
        border-bottom: none;
    }

    .dropdown-item strong {
        color: #0062A3;
    }

    /* FIXED POPUP */
    :global(.fixed-popup-container) {
        position: absolute;
        top: 10px;
        right: 50px;
        z-index: 2;
        background: white;
        border-radius: 8px;
        box-shadow: 0 2px 12px rgba(0, 0, 0, 0.2);
        max-width: 250px;
    }

     :global(.fixed-popup-close) {
        position: absolute;
        top: 4px;
        right: 8px;
        background: none;
        border: none;
        font-size: 24px;
        font-weight: 800;
        color: #FFF;
        cursor: pointer;
        padding: 0;
        width: 24px;
        height: 24px;
        line-height: 20px;
        font-weight: 300;
        z-index: 5;
    }

    .fixed-popup-close:hover {
        color: #231F20;
    }

    .fixed-popup {
        /* padding-right: 20px; */
    }
</style>