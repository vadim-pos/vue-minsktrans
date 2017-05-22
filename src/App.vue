<template>
    <div id="schedule-app">
        <button class="menu-trigger" @click="showMenu=!showMenu">
            {{showMenu ? '&#10006;' : '&#9776;'}}
        </button>
        <h1 class="title">{{ title }}</h1>

        <div class="container">
            <div class="menu" v-if="showMenu">
                <div class="menu-group" v-for="transport in availiableTransport">
                    <input :id="transport.type" type="radio" name="transport"
                    :value="transport.type"
                    @change="switchTransportType(transport)"
                    v-model="activeTransportType">
                    <label :for="transport.type">{{transport.name}}</label>
                </div>
            </div>
            <div class="routes">
                <div class="routes-preloader" v-if="!filteredRoutes.length"></div>
                <ul class="routes-list">
                    <li class="routes-item" v-for="route in filteredRoutes">
                        <a href="#" class="routes-link" @click.prevent="renderRouteStops(route, $event)">
                            <span class="routes-num">{{route.RouteNum}}</span>
                            <span class="routes-name">{{route.RouteName}}</span>
                        </a>
                    </li>
                </ul>
            </div>

            <div class="info">
                <div class="info-message-wrap" v-if="infoMessageIsActive">
                    <p class="info-message">{{ infoMessage }}</p>
                    <button class="info-message-btn" @click="infoMessageIsActive = false">Ок</button>
                </div>
                <div class="map" ref="map"></div>
            </div>

        </div>
    </div>
</template>

<script>
export default {
    http: {
        root: 'https://cors-anywhere.herokuapp.com/http://www.minsktrans.by/city/minsk'
    },

    data: () => ({
        /* type of transport for which is needed to dispaly routes */
        activeTransportType: 'bus',
        /* availiable transport types to choose from */
        availiableTransport: [
            { type: 'bus', name: 'Автобусы' },
            { type: 'metro', name: 'Метро' },
            { type: 'tram', name: 'Трамваи' },
            { type: 'trol', name: 'Троллейбусы' }
        ],
        
        allRoutes: [],
        filteredRoutes: [],
        stops: [],

        map: null,
        /* stores google map info-window instance */
        mapInfoWindow: null,
        markers: [],
        /* info message contains errors and other information for user. Overlays map on the screen */
        infoMessageIsActive: false,
        infoMessage: '',

        activeRouteLink: null,
        showMenu: false
        }),
    
    created() {
        /* prepare (fetch, parse, filter) and set data */
        this.$http.get('routes.txt').then(res => {
                this.allRoutes = this.parseTextData(res.data);
                this.filteredRoutes = this.filterRoutesByType(this.allRoutes, this.activeTransportType);
            }, err => this.showInfoMessage('Не удалось получить данные c сервера')
        );
        this.$http.get('stops.txt').then(res => {
                this.stops = this.parseTextData(res.data);
            }, err => this.showInfoMessage('Не удалось получить данные с сервера')
        );
        /* initialize google map */
        google.maps.event.addDomListener(window, 'load', this.initMap);
    },

    computed: {
        /* application heading */
        title() {
            let routesStr = '';
            let cityStr = 'г. Минска';

            if (this.activeTransportType === 'bus') { routesStr = 'Автобусные маршруты' }
            else if (this.activeTransportType === 'metro') { routesStr = 'Маршруты метро' }
            else if (this.activeTransportType === 'tram') { routesStr = 'Трамвайные маршруты' }
            else if (this.activeTransportType === 'trol') { routesStr = 'Троллейбусные маршруты' }

            return `${routesStr} ${cityStr}`;
        }
    },

    methods: {
        /* parse text into array of objects */
        parseTextData(text) {
            const rowsArr = text.split('\n');
            const keysArr = rowsArr[0].split(';');
            const resultArr = [];

            for (let i = 1; i < rowsArr.length; i++) {
                let parsedItem = {};

                rowsArr[i].split(';').forEach((value, i) => {
                    parsedItem[keysArr[i]] = value;
                });

                resultArr.push(parsedItem);
            }
            return resultArr;
        },

        filterRoutesByType(routes, type) {
            let startIndex = null;
            let endIndex = null;
            let max = routes.length;
            
            /* get startIndex */
            for (let i = 0; i < max; i++) {
                if (routes[i].Transport === type) {
                    startIndex = i;
                    break;
                }
            }
            /* get endIndex */
            for (let i = startIndex + 1; i < max; i++) {
                if (routes[i].Transport !== '') {
                    endIndex = i;
                    break;
                }
            }
            
            return routes
                /* get all routes of this type */
                .slice(startIndex, endIndex)
                /* get only forward (A>B) routes with defined route numbers */
                .filter(route => route.RouteType === 'A>B' && route.RouteNum !== '');
        },

        switchTransportType(transportType) {
            this.filteredRoutes = this.filterRoutesByType(this.allRoutes, this.activeTransportType);
            this.removeMarkers();
            if (this.activeRouteLink) { this.activeRouteLink.classList.remove('active'); }
            this.showMenu = false;
        },

        renderRouteStops(route, e) {
            /* deactivate previous active link */
            if (this.activeRouteLink) { this.activeRouteLink.classList.remove('active'); }
            /* remove existing markers */
            this.removeMarkers();
            /* hide info messages */
            this.infoMessageIsActive = false;
            
            /* activate link and save link elemnt in data */
            if (e.currentTarget.classList.contains('routes-link')) {
                let activeLink = e.currentTarget;
                activeLink.classList.add('active');
                this.activeRouteLink = activeLink;
            }
            /* get required markers data */
            const markersSettings = this.getRouteMarkersSettings(route);

            /* if no data exist show info message */
            if (!markersSettings) {
                return this.showInfoMessage('В настоящее время информация по данному маршруту отсутствует');
            }
            /* add new markers to google map */
            markersSettings.forEach((markerData, i) => this.addMarker(markerData));
        },
        
        /* get necessary data for markers creation */
        getRouteMarkersSettings(route) {
            if (route.RouteStops === '') { return null; }

            const stopsIDs = route.RouteStops.split(',');
            let stopsArr = [];

            stopsIDs.forEach(id => {
                let index = this.getStopIndexById(id);
                let currentStop = this.stops[index];
                let stopObj = {};
                
                /* get converted coordinates */
                stopObj.lat = this.stringToGPS(currentStop.Lat);
                stopObj.lng = this.stringToGPS(currentStop.Lng);
                
                /* if name prop does not exist, find root stop element */
                if (this.stops[index].Name === '') {
                    let i = index;

                    do { currentStop = this.stops[i--]; }
                    while (currentStop.Name === '')
                }
                
                /* get stop name */
                stopObj.title = currentStop.Name;

                stopsArr.push(stopObj);
            });
            
            return(stopsArr);
        },

        /* get specific stop in array of stops by it's id */
        getStopIndexById(id) {
            for (let i = 0; i < this.stops.length; i++) {
                if (this.stops[i].ID === id) {
                    return i;
                }
            }
        },
        
        /* configure and add new marker to google map */
        addMarker(markerData) {
            let {lat, lng, title} = markerData;

            let marker = new google.maps.Marker({
                position: new google.maps.LatLng(lat, lng),
                map: this.map,
                title,
                label: {
                    text: 'A',
                    color: '#ffffff',
                    fontWeight: 'bold'
                }
            });
            
            google.maps.event.addListener(marker, 'click', () => {
                const infoContent = `<div class="iw-content">${title}</div>`;
                this.mapInfoWindow.close(); // close previous
                this.mapInfoWindow.setContent(infoContent);
                this.mapInfoWindow.open(this.map, marker);
            });

            this.markers.push(marker);
        },

        removeMarkers() {
            if (this.markers.length) {
                this.markers.forEach(marker => marker.setMap(null));
                this.markers = [];
            }
        },

        stringToGPS(str) {
            return parseFloat(str.slice(0,2) + '.' + str.slice(2));
        },

        showInfoMessage(message) {
            this.infoMessage = message;
            this.infoMessageIsActive = true;
        },

        initMap() {
            let mapCenter = null;
            const mapOptions = {
                zoom: 12,
                center: new google.maps.LatLng(53.9045326, 27.5744375), // Minsk
                streetViewControl: false,
                mapTypeControl: false,
                styles: [{"featureType":"landscape.man_made","elementType":"geometry","stylers":[{"color":"#f7f1df"}]},{"featureType":"landscape.natural","elementType":"geometry","stylers":[{"color":"#d0e3b4"}]},{"featureType":"landscape.natural.terrain","elementType":"geometry","stylers":[{"visibility":"on"}]},{"featureType":"poi","elementType":"labels","stylers":[{"visibility":"on"}]},{"featureType":"poi.business","elementType":"all","stylers":[{"visibility":"on"}]},{"featureType":"poi.medical","elementType":"geometry","stylers":[{"color":"#fbd3da"}]},{"featureType":"poi.park","elementType":"geometry","stylers":[{"color":"#bde6ab"}]},{"featureType":"road","elementType":"geometry.stroke","stylers":[{"visibility":"on"}]},{"featureType":"road","elementType":"labels","stylers":[{"visibility":"on"}]},{"featureType":"road.highway","elementType":"geometry.fill","stylers":[{"color":"#fd997c"}]},{"featureType":"road.highway","elementType":"geometry.stroke","stylers":[{"color":"#efd151"}]},{"featureType":"road.arterial","elementType":"geometry.fill","stylers":[{"color":"#fd997c"}]},{"featureType":"road.local","elementType":"geometry.fill","stylers":[{"color":"#fd997c"}]},{"featureType":"transit.station.airport","elementType":"geometry.fill","stylers":[{"color":"#cfb2db"}]},{"featureType":"water","elementType":"geometry","stylers":[{"color":"#a2daf2"}]}]
            };

            this.map = new google.maps.Map(this.$refs.map, mapOptions);
            this.mapInfoWindow = new google.maps.InfoWindow();
            
            mapCenter = this.map.getCenter();

            google.maps.event.addDomListener(window, 'resize', () => {
                this.map.setCenter(mapCenter);
            });
            google.maps.event.addListener(this.map, 'click', () => {
                this.mapInfoWindow.close();
            });
        }
    }
}
</script>

<style lang="scss">
    @import 'scss/styles.scss';
</style>
