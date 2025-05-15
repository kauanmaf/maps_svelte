<script>
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    import { onMount } from "svelte";
    import * as d3 from 'd3';

    mapboxgl.accessToken = "pk.eyJ1Ijoia2F1YW5tYWYiLCJhIjoiY21hcGd6ZmRrMGk3cTJscHZtYjh1Znp4eCJ9.kD1iAdL1_lTda-IhsnyLKQ";

    let map;
    let stations = [];
    let mapViewChanged = 0;
    let trips = [];
    
    // Escala do raio dos círculos
    $: radiusScale = d3.scaleSqrt()
        .domain([0, d3.max(stations, d => d.totalTraffic) || 0])
        .range([0, 25]);


    // Atualiza a visualização sempre que o mapa for movido
    $: map?.on("move", () => mapViewChanged++);

    // Converte coordenadas geográficas em coordenadas de tela
    function getCoords(station) {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let { x, y } = map.project(point);
        return { cx: x, cy: y };
    }

    // Carrega os dados das estações
    async function loadStationData() {
        try {
            const csvUrl = 'https://vis-society.github.io/labs/8/data/bluebikes-stations.csv';
            const data = await d3.csv(csvUrl);
            stations = data.map(station => ({
                id: station.Number,
                name: station.NAME,
                Lat: +station.Lat,
                Long: +station.Long,
            }));
        } catch (error) {
            console.error('Erro ao carregar estações:', error);
        }
    }

    // Carrega os dados de viagens
    async function loadStationDemand() {
        try {
            const csvUrl = 'https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv';
            const data = await d3.csv(csvUrl);
            trips = data.map(trip => ({
                id: trip.ride_id,
                name: trip.NAME,
                started_at: new Date(trip.started_at),
                ended_at: new Date(trip.ended_at),
                start_station_id: trip.start_station_id,
                end_station_id: trip.end_station_id
            }));
        } catch (error) {
            console.error('Erro ao carregar viagens:', error);
        }
    }

    // Inicializa o mapa
    async function initMap() {
        map = new mapboxgl.Map({
            container: 'map',
            center: [-71.09415, 42.36027],
            zoom: 12,
            style: "mapbox://styles/kauanmaf/cmaphkifu000b01sc5ukd179h",
        });

        await new Promise(resolve => map.on("load", resolve));

        map.addSource("boston_route", {
            type: "geojson",
            data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
        });

        map.addSource("cambridge_route", {
            type: "geojson",
            data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
        });

        map.addLayer({
            id: "boston_bike_routes",
            type: "line",
            source: "boston_route",
            paint: {
                'line-color': 'green',
                'line-width': 3,
                'line-opacity': 0.5
            },
        });

        map.addLayer({
            id: "cambridge_bike_routes",
            type: "line",
            source: "cambridge_route",
            paint: {
                'line-color': 'green',
                'line-width': 3,
                'line-opacity': 0.5
            },
        });
    }

    // Executa ao montar o componente
    onMount(async () => {
        await initMap();
        await loadStationData();
        await loadStationDemand();

        const departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
        const arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);

        stations = stations.map(station => {
            const id = station.id;
            station.arrivals = arrivals.get(id) ?? 0;
            station.departures = departures.get(id) ?? 0;
            station.totalTraffic = station.arrivals + station.departures;
            return station;
        });
    });
</script>


<h1>Bikes</h1>

<div id="map">
	<svg>
        {#key mapViewChanged}
            {#each stations as station}
                <circle
                    {...getCoords(station)}
                    r={radiusScale(station.totalTraffic)}
                    fill="steelblue"
                    fill-opacity="0.7"
                    stroke="white"
                    stroke-width="1"
                />
            {/each}
        {/key}
    </svg>
</div>

<style>
@import url("$lib/global.css");
</style>