<template>
  <div>
    <!-- display map -->
    <div id="map" ref="mapDiv" style="width: 80%; height: 50vh;"></div>
    
    <button @click="getCurrentLocation" class="btn btn-outline-dark">Get Current Location</button>

    <!-- search module -->
    <div id="search" class="d-flex align-items-center justify-content-center">
      <label>Name of a location:</label>
      <input type="text" v-model.trim="searchQuery" @keyup.enter="searchLocation" />
      <button @click="searchLocation" class="btn btn-outline-dark">Search</button>
    </div>
 
    <!-- table with pagination -->
    <table class="table">
      <thead>
        <tr>
          <th><input type="checkbox" v-model="selectAll" /></th>
          <th>Location Name</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="location in paginatedLocations" :key="location.id">
          <td>
            <input type="checkbox" v-model="location.selected" />
          </td>
          <td>{{ location.name }}</td>
        </tr>
      </tbody>
    </table>

    <div id="table-buttons">
      <button @click="previousPage" class="btn btn-outline-secondary">Previous</button>
      <button @click="deleteLocation" class="btn btn-outline-danger">Delete</button>
      <button @click="nextPage" class="btn btn-outline-secondary">Next</button>
    </div>

    <!-- Display the time zone and local time -->
    <div>
      <p>Time Zone: {{ timeZone }}</p>
      <p>Local Time: {{ localTime }}</p>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from "vue"
import { Loader } from "@googlemaps/js-api-loader"
import { useGeolocation } from "./useGeolocation"

const GOOGLE_MAPS_API_KEY = 'your Google Maps api key'

export default {
  setup() {
    let map
    let service
    let locationId = 0
    const { coords } = useGeolocation()
    const currentLocation = computed(() => {
      return {
        lat: coords.value.latitude,
        lng: coords.value.longitude
      }
    })
    const mapDiv = ref(null)
    
    const searchQuery = ref('')
    const allLocations = ref([])

    const markers = []

    const currentPage = ref(0)
    const timeZone = ref('')
    const localTime = ref('')

    const loader = new Loader({
      apiKey: GOOGLE_MAPS_API_KEY,
      version: "weekly",
    })

    onMounted(async () => {
      await loader.load()
      map = new google.maps.Map(mapDiv.value, {
        center: currentLocation.value,
        zoom: 8
      })
      const {PlacesService} = await google.maps.importLibrary("places")
      service = new PlacesService(map)
    })

    const paginatedLocations = computed(() => {
      let start = currentPage.value * 10
      return allLocations.value.slice(start, start + 10)
    })

    const getCurrentLocation = () => {
      // useGeolocation is already watching current position and updating coords
      // and currentLocation will be updated accordingly since it's a computed value from coords
      map.setCenter(currentLocation.value)
    }

    const searchLocation = () => {
      const request = {
        query: searchQuery.value,
        fields: ['name', 'geometry']
      }
      service.findPlaceFromQuery(request, (results, status) => {
        if (status === google.maps.places.PlacesServiceStatus.OK) {
          let location
          const foundLocationIndex = allLocations.value.findIndex(location => location.name === results[0].name)
          if ( foundLocationIndex === -1) {
            const marker = new google.maps.Marker({
              position: results[0].geometry.location,
              map
            })

            const newLocation = {
              name: results[0].name,
              id: locationId++,
              lat: results[0].geometry.location.lat(),
              lng: results[0].geometry.location.lng(),
              selected: false
            }

            markers.push({id: newLocation.id, marker})
            allLocations.value.push(newLocation)
            location = newLocation
          } else {
            location = allLocations.value[foundLocationIndex]
          }
          console.log(results[0].geometry.location.lat())
          fetchTimeZone(location)
          map.setCenter(results[0].geometry.location)
        }
      })
    }

    const deleteLocation = () => {
      for(let location of allLocations.value) {
        if (location.selected) {
          const foundIndex = markers.findIndex(marker => marker.id === location.id)
          markers[foundIndex].marker.setMap(null)
          markers.splice(foundIndex, 1)
        }
      }
      allLocations.value = allLocations.value.filter((location) => {
        return !location.selected
      })
    }

    const previousPage = () => {
      if (currentPage.value > 0) currentPage.value--
    }

    const nextPage = () => {
      if ((currentPage.value + 1) * 10 < allLocations.value.length) currentPage.value++
    }
    
    const selectAll = computed({
      get: () => allLocations.value.every(location => location.selected),
      set: (val) => allLocations.value.forEach(location => location.selected = val)
    })

    const fetchTimeZone = (location) => {
      fetch(`https://maps.googleapis.com/maps/api/timezone/json?location=${location.lat},${location.lng}&timestamp=${Math.round(new Date().getTime() / 1000)}&key=${GOOGLE_MAPS_API_KEY}`)
        .then(response => response.json())
        .then(data => {
          console.log(data);
          timeZone.value = data.timeZoneName
          const rawOffset = data.rawOffset
          const dstOffset = data.dstOffset
          localTime.value = new Date(Date.now() + (rawOffset + dstOffset) * 1000).toLocaleString()
        })
        .catch(error => console.error('Error fetching time zone:', error))
    }

    return {
      mapDiv,
      currentLocation,
      searchQuery,
      allLocations,
      currentPage,
      timeZone,
      localTime,
      paginatedLocations,
      getCurrentLocation,
      searchLocation,
      deleteLocation,
      previousPage,
      nextPage,
      selectAll
    }
  }
}

</script>

<style>
div#map {
  margin: 20px auto;
}
div#search {
  margin: 40px auto;
}
input {
  max-width: 400px;
}
table {
  margin: 20px auto;
  max-width: 600px;
}
div#table-buttons button {
  margin: 0 40px 40px;
}
</style>
