<script setup>
import { reactive, ref, onMounted } from 'vue'

let incident_data = ref([]);
let crime_url = ref('');
let neighborhood_names = ref([]);
let dialog_err = ref(false);
let search_location = ref('');
let map = reactive(
    {
        leaflet: null,
        center: {
            lat: 44.955139,
            lng: -93.102222,
            address: ''
        },
        zoom: 12,
        bounds: {
            nw: {lat: 45.008206, lng: -93.217977},
            se: {lat: 44.883658, lng: -92.993787}
        },
        neighborhood_markers: [
            {location: [44.942068, -93.020521], marker: null},
            {location: [44.977413, -93.025156], marker: null},
            {location: [44.931244, -93.079578], marker: null},
            {location: [44.956192, -93.060189], marker: null},
            {location: [44.978883, -93.068163], marker: null},
            {location: [44.975766, -93.113887], marker: null},
            {location: [44.959639, -93.121271], marker: null},
            {location: [44.947700, -93.128505], marker: null},
            {location: [44.930276, -93.119911], marker: null},
            {location: [44.982752, -93.147910], marker: null},
            {location: [44.963631, -93.167548], marker: null},
            {location: [44.973971, -93.197965], marker: null},
            {location: [44.949043, -93.178261], marker: null},
            {location: [44.934848, -93.176736], marker: null},
            {location: [44.913106, -93.170779], marker: null},
            {location: [44.937705, -93.136997], marker: null},
            {location: [44.949203, -93.093739], marker: null}
        ]
    }
);

// Vue callback for once <template> HTML has been added to web page
onMounted(() => {
    // Create Leaflet map (set bounds and valied zoom levels)
    map.leaflet = L.map('leafletmap').setView([map.center.lat, map.center.lng], map.zoom);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
        minZoom: 11,
        maxZoom: 18
    }).addTo(map.leaflet);
    map.leaflet.setMaxBounds([[44.883658, -93.217977], [45.008206, -92.993787]]);

    // Get boundaries for St. Paul neighborhoods
    let district_boundary = new L.geoJson();
    district_boundary.addTo(map.leaflet);
    fetch('data/StPaulDistrictCouncil.geojson')
    .then((response) => {
        return response.json();
    })
    .then((result) => {
        result.features.forEach((value) => {
            district_boundary.addData(value);
        });
    })
    .catch((error) => {
        console.log('Error:', error);
    });
});

// Will calculate the amount of crime that happens in each neighborhood, used in addMarkers()
function crimeAmountPerNeighboorhood(incidents){
    console.log(map.neighborhood_markers.length);
    for(let k = 0; k < map.neighborhood_markers.length; k++){
        map.neighborhood_markers[k].crimeAmount = 0;
    }

    for(let number of incidents.value){
        let index = number.neighborhood_number - 1;
        map.neighborhood_markers[index].crimeAmount+= 1;
    }
    console.log(map.neighborhood_markers[0].crimeAmount);
    addMarkers();
}

// This function will add markers to the map
function addMarkers() {
    console.log(map.neighborhood_markers[1].crimeAmount);
    for(let i = 0; i < map.neighborhood_markers.length; i++){ //let locations in map.neighborhood_markers){
        let lat = map.neighborhood_markers[i].location[0];
        let long = map.neighborhood_markers[i].location[1];
        map.neighborhood_markers[i].marker = L.marker([lat, long]).addTo(map.leaflet).bindPopup(String(map.neighborhood_markers[i].crimeAmount));
    }
}

// Function to get neighborhood names
function getNeighboorhoodNames() {
    /*
    let base_url = crime_url.value.split("/incidents"); //the host and port number will be located at base_url[0]
    console.log(base_url[0] + '/neighborhoods');
    */
    fetch(crime_url.value + '/neighborhoods')
    .then((response_neighborhood) => {
        return response_neighborhood.json();
    })
    .then((neighborhood_data) => {
        neighborhood_names.value = neighborhood_data;
        //console.log(neighborhood_names.value);
    })
    .catch((err) => {
        console.log(err);
    })
}

async function repopulate_incidents() {
  let lat, lng

  if (/^\s*-?\d+(\.\d+)?\s*,\s*-?\d+(\.\d+)?\s*$/.test(search_location.value)) {
    const parts = search_location.value.split(',')
    lat = Number(parts[0])
    lng = Number(parts[1])
  }

  else {
    const url = `https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(search_location.value)}`
    const res = await fetch(url)
    const data = await res.json()

    if (!data.length) {
      alert('Location not found')
      return
    }

    lat = Number(data[0].lat)
    lng = Number(data[0].lon)
  }

  map.leaflet.setView([lat, lng], 14)
}


// FUNCTIONS
// Function called once user has entered REST API URL
function initializeCrimes() {
    // TODO: get code and neighborhood data
    //       get initial 1000 crimes
    console.log(crime_url.value + '/incidents');
    let options = {
        method: 'GET',
    };
    fetch(crime_url.value + '/incidents', options)
    .then((response) => {
        console.log(response);
        return response.json();
    })
    .then((api_data) => {
        incident_data.value = api_data;
        //console.log(incident_data.values[i].neighborhood_number)
        crimeAmountPerNeighboorhood(incident_data);
    })
    .catch((err) => {
        console.log(err);
    })
}

// Function called when user presses 'OK' on dialog box
function closeDialog() {
    let dialog = document.getElementById('rest-dialog');
    let url_input = document.getElementById('dialog-url');
    if (crime_url.value !== '' && url_input.checkValidity()) {
        dialog_err.value = false;
        dialog.close();
        getNeighboorhoodNames();
        initializeCrimes();
    }
    else {
        dialog_err.value = true;
    }
}

function remove_incident(incident) {
    let options = {
        method: 'DELETE',
        body: JSON.stringify(incident)
    };
    fetch(crime_url.value + '/remove-incident', options)
        .then( res => {
            console.log(res);
        })
        .catch( err => {
            console.log(err);
        });
}
</script>

<template>
    <dialog id="rest-dialog" open>
        <h1 class="dialog-header">St. Paul Crime REST API</h1>
        <label class="dialog-label">URL: </label>
        <input id="dialog-url" class="dialog-input" type="url" v-model="crime_url" placeholder="http://localhost:8000" />
        <p class="dialog-error" v-if="dialog_err">Error: must enter valid URL</p>
        <br/>
        <button class="button" type="button" @click="closeDialog">OK</button>
    </dialog>
    <div class="grid-container ">
        <div class="grid-x grid-padding-x">
            <div id="leafletmap" class="cell auto"></div>
        </div>
    </div>
    <label>Search: </label> <input type="text" v-model="search_location"/> <button type="button" @click="repopulate_incidents">Go</button>
    <table>
        <thead>
          <tr>
              <th>Case Number</th>
              <th>Incident</th>
              <th>Neighborhood Name</th>
              <th>Block</th>
              <th>Date</th>
              <th>Time</th>
              <th>Delete</th>
          </tr>
        </thead>
        <tbody>
            <tr v-for="(item) in incident_data">
                <td>{{ item.case_number }}</td>
                <td>{{ item.incident }}</td>
                <td>{{ neighborhood_names.find(nb => nb.neighborhood_number === item.neighborhood_number).neighborhood_name }}</td>
                <td>{{ item.block }}</td>
                <td>{{ item.date }}</td>
                <td>{{ item.time }}</td>
                <td>
                    <button type="button" @click="remove_incident(item)">Delete</button>
                </td>
            </tr>
        </tbody>
    </table>
</template>

<style scoped>
#rest-dialog {
    width: 20rem;
    margin-top: 1rem;
    z-index: 1000;
}

#leafletmap {
    height: 500px;
}

.dialog-header {
    font-size: 1.2rem;
    font-weight: bold;
}

.dialog-label {
    font-size: 1rem;
}

.dialog-input {
    font-size: 1rem;
    width: 100%;
}

.dialog-error {
    font-size: 1rem;
    color: #D32323;
}
</style>
