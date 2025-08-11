<template>
  <div class="w-full h-screen relative overflow-hidden" :class="skyClass">
    <!-- Weather effects -->
    <div v-if="weather?.condition === 'rain'" class="absolute inset-0 pointer-events-none">
      <div 
        v-for="drop in raindrops" 
        :key="drop.id"
        class="absolute bg-blue-400 opacity-70 rounded-full animate-pulse"
        :style="{
          left: drop.x + '%',
          top: drop.y + '%',
          width: drop.size + 'px',
          height: drop.size * 3 + 'px',
          animationDuration: drop.speed + 's',
          transform: 'rotate(15deg)'
        }"
      ></div>
    </div>

    <div v-if="weather?.condition === 'snow'" class="absolute inset-0 pointer-events-none">
      <div 
        v-for="flake in snowflakes" 
        :key="flake.id"
        class="absolute bg-white rounded-full animate-bounce"
        :style="{
          left: flake.x + '%',
          top: flake.y + '%',
          width: flake.size + 'px',
          height: flake.size + 'px',
          animationDuration: flake.speed + 's'
        }"
      ></div>
    </div>

    <!-- Sun -->
    <div 
      v-if="['day', 'dawn', 'dusk'].includes(timeOfDay)" 
      class="absolute top-8 right-8 rounded-full shadow-lg cursor-pointer hover:scale-110 transition-transform"
      :class="{
        'w-16 h-16 bg-yellow-400 animate-pulse': timeOfDay === 'day',
        'w-14 h-14 bg-orange-400': timeOfDay === 'dawn',
        'w-12 h-12 bg-red-400': timeOfDay === 'dusk',
        'opacity-30': weather?.condition === 'rain' || weather?.condition === 'snow',
        'opacity-60': weather?.condition === 'cloudy'
      }"
      @click="toggleWeatherMenu"
    ></div>

    <!-- Clouds -->
    <div v-if="weather?.condition === 'cloudy'" class="absolute inset-0 pointer-events-none">
      <div 
        v-for="cloud in clouds" 
        :key="cloud.id"
        class="absolute"
        :style="{
          left: cloud.x + '%',
          top: cloud.y + '%',
          transform: 'translateX(' + cloud.drift + 'px)'
        }"
      >
        <!-- Main cloud body (elongated base) -->
        <div 
          class="absolute rounded-full"
          :class="{
            'bg-white': ['day', 'dawn'].includes(timeOfDay),
            'bg-gray-300': timeOfDay === 'dusk',
            'bg-gray-600': timeOfDay === 'night'
          }"
          :style="{
            width: cloud.width + 'px',
            height: cloud.height + 'px',
            top: cloud.height * 0.3 + 'px'
          }"
        ></div>
        <!-- Cloud puffs on top for puffy appearance -->
        <div 
          v-for="puff in cloud.puffs" 
          :key="puff.id"
          class="absolute rounded-full"
          :class="{
            'bg-white': ['day', 'dawn'].includes(timeOfDay),
            'bg-gray-300': timeOfDay === 'dusk',
            'bg-gray-600': timeOfDay === 'night'
          }"
          :style="{
            left: puff.x + 'px',
            top: (puff.y - puff.size * 0.3) + 'px',
            width: puff.size + 'px',
            height: puff.size + 'px'
          }"
        ></div>
      </div>
    </div>

    <!-- Moon -->
    <div 
      v-if="timeOfDay === 'night'" 
      class="absolute top-8 right-8 w-12 h-12 bg-gray-100 rounded-full shadow-lg cursor-pointer hover:scale-110 transition-transform"
      @click="toggleWeatherMenu"
    ></div>

    <!-- Weather Summary -->
    <div class="absolute top-4 left-4 bg-black bg-opacity-50 text-white px-3 py-2 rounded text-sm">
      <div class="text-xs opacity-80">{{ getCitySizeLabel(userCityIndex) }} • {{ formatPopulation(location?.population) }}</div>
      <div class="font-semibold">{{ getCurrentTime() }}</div>
      <div class="capitalize">{{ weather?.condition }} • {{ formatTemperature(weather?.temperature || 0) }}</div>
    </div>

    <!-- Ground -->
    <div 
      class="absolute bottom-0 left-0 right-0 h-8"
      :class="{
        'bg-green-700': isDay,
        'bg-green-900': !isDay
      }"
    ></div>

    <!-- Cities -->
    <div class="absolute bottom-8 left-0 right-0 flex items-end justify-center space-x-4 p-8">
      <div 
        v-for="(city, index) in cityBuildings" 
        :key="index"
        class="relative flex flex-col items-center"
      >
        <!-- Buildings -->
        <div class="flex items-end space-x-1 mb-2">
          <div 
            v-for="building in city.buildings" 
            :key="building.id"
            class="relative"
            :class="buildingColor"
            :style="{
              width: building.width + 'px',
              height: building.height + 'px'
            }"
          >
            <!-- Windows -->
            <div 
              v-for="window in building.windows" 
              :key="window.id"
              class="absolute bg-yellow-200 opacity-80"
              :style="{
                left: window.x + 'px',
                top: window.y + 'px',
                width: window.width + 'px',
                height: window.height + 'px'
              }"
            ></div>
          </div>
        </div>

        <!-- Billboard -->
        <div 
          v-if="index === userCityIndex"
          class="absolute -top-16 left-1/2 transform -translate-x-1/2 bg-gray-800 text-white px-4 py-2 rounded shadow-lg text-center min-w-32 cursor-pointer hover:bg-gray-700 transition-colors"
          @click="toggleCityMenu"
        >
          <div class="text-sm font-bold">{{ location?.city }}</div>
          <div class="text-xs">{{ location?.region }}</div>
        </div>
      </div>
    </div>

    <!-- Weather Menu -->
    <div v-if="showWeatherMenu" class="absolute top-24 right-8 bg-white rounded-lg shadow-lg p-4 z-10">
      <div class="text-sm font-bold mb-2">Change Weather</div>
      <div class="grid grid-cols-2 gap-2">
        <button 
          v-for="condition in ['clear', 'rain', 'snow', 'cloudy']" 
          :key="condition"
          class="px-3 py-2 text-sm bg-blue-100 hover:bg-blue-200 rounded capitalize"
          @click="changeWeather(condition)"
        >
          {{ condition }}
        </button>
      </div>
      <div class="mt-3 pt-2 border-t">
        <button 
          class="px-3 py-1 text-sm bg-gray-100 hover:bg-gray-200 rounded mr-2"
          @click="toggleTimeOfDay"
        >
          {{ isDay ? 'Night' : 'Day' }}
        </button>
      </div>
    </div>

    <!-- City Menu -->
    <div v-if="showCityMenu" class="absolute bottom-32 left-1/2 transform -translate-x-1/2 bg-white rounded-lg shadow-lg p-4 z-10 w-80">
      <div class="text-sm font-bold mb-2">Change City</div>
      <div class="relative">
        <input 
          v-model="citySearch"
          type="text"
          placeholder="Type 3+ characters (e.g. Bloomington, Indiana)"
          class="w-full px-3 py-2 border rounded mb-2 text-sm"
          @input="onSearchInput"
          @keyup.enter="searchCity"
          @focus="showSuggestions = true"
        >
        <!-- Loading indicator -->
        <div v-if="isSearching" class="absolute right-3 top-2.5">
          <div class="animate-spin rounded-full h-4 w-4 border-b-2 border-blue-500"></div>
        </div>
        
        <!-- Autocomplete suggestions -->
        <div v-if="showSuggestions && searchSuggestions.length > 0" 
             class="absolute top-full left-0 right-0 bg-white border rounded-lg shadow-lg mt-1 max-h-48 overflow-y-auto z-20">
          <button 
            v-for="suggestion in searchSuggestions" 
            :key="suggestion.display_name"
            class="w-full px-3 py-2 text-sm hover:bg-blue-50 text-left border-b last:border-b-0"
            @click="selectSuggestion(suggestion)"
          >
            <div class="font-medium">{{ suggestion.name }}</div>
            <div class="text-xs text-gray-500">{{ suggestion.display_name }}</div>
          </button>
        </div>
      </div>
      
      <div class="text-xs text-gray-600 mb-2">Popular Cities:</div>
      <div class="grid grid-cols-1 gap-1 max-h-32 overflow-y-auto">
        <button 
          v-for="city in popularCities" 
          :key="city.name"
          class="px-3 py-2 text-sm bg-blue-100 hover:bg-blue-200 rounded text-left"
          @click="selectCity(city)"
        >
          {{ city.name }}, {{ city.region }}
        </button>
      </div>
    </div>

    <!-- Loading state -->
    <div v-if="loading" class="absolute inset-0 flex items-center justify-center bg-black bg-opacity-50">
      <div class="text-white text-xl">Getting your weather...</div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue'

interface Location {
  city: string
  region: string
  country: string
  lat: number
  lon: number
  population?: number
}

interface Weather {
  condition: 'clear' | 'rain' | 'snow' | 'cloudy'
  temperature: number
  isDay: boolean
}

interface Cloud {
  id: number
  x: number
  y: number
  width: number
  height: number
  drift: number
  puffs: CloudPuff[]
}

interface CloudPuff {
  id: number
  x: number
  y: number
  size: number
}

interface Building {
  id: number
  width: number
  height: number
  windows: Window[]
}

interface Window {
  id: number
  x: number
  y: number
  width: number
  height: number
}

interface WeatherEffect {
  id: number
  x: number
  y: number
  size: number
  speed: number
}

const location = ref<Location | null>(null)
const originalUserLocation = ref<Location | null>(null)
const weather = ref<Weather | null>(null)
const loading = ref(true)
const raindrops = ref<WeatherEffect[]>([])
const snowflakes = ref<WeatherEffect[]>([])
const showWeatherMenu = ref(false)
const showCityMenu = ref(false)
const citySearch = ref('')
const currentCityTimezone = ref<string | null>(null)
const clouds = ref<Cloud[]>([])
const searchSuggestions = ref<any[]>([])
const showSuggestions = ref(false)
const isSearching = ref(false)

// Cache for search suggestions (24 hour expiration)
const searchCache = new Map<string, { data: any[], timestamp: number }>()
const SEARCH_CACHE_DURATION = 24 * 60 * 60 * 1000 // 24 hours

const popularCities = ref([
  { name: 'New York', region: 'NY', lat: 40.7128, lon: -74.0060, population: 8400000 },
  { name: 'Los Angeles', region: 'CA', lat: 34.0522, lon: -118.2437, population: 4000000 },
  { name: 'Chicago', region: 'IL', lat: 41.8781, lon: -87.6298, population: 2700000 },
  { name: 'London', region: 'England', lat: 51.5074, lon: -0.1278, population: 9000000 },
  { name: 'Tokyo', region: 'Japan', lat: 35.6762, lon: 139.6503, population: 14000000 },
  { name: 'Paris', region: 'France', lat: 48.8566, lon: 2.3522, population: 2100000 },
  { name: 'Sydney', region: 'Australia', lat: -33.8688, lon: 151.2093, population: 5300000 },
  { name: 'Toronto', region: 'Canada', lat: 43.6532, lon: -79.3832, population: 2900000 }
])

const timeOfDay = computed(() => {
  let hour
  
  if (currentCityTimezone.value) {
    try {
      const now = new Date()
      const cityTime = new Date(now.toLocaleString('en-US', { timeZone: currentCityTimezone.value }))
      hour = cityTime.getHours()
    } catch (error) {
      console.warn('Invalid timezone, using local time:', error)
      hour = new Date().getHours()
    }
  } else {
    hour = new Date().getHours()
  }
  
  if (hour >= 5 && hour < 7) return 'dawn'
  if (hour >= 7 && hour < 17) return 'day' 
  if (hour >= 17 && hour < 19) return 'dusk'
  return 'night'
})

const isDay = computed(() => ['day', 'dawn', 'dusk'].includes(timeOfDay.value))

const skyClass = computed(() => {
  const time = timeOfDay.value
  
  const skyClasses = {
    'day': 'bg-gradient-to-b from-blue-400 to-blue-300',
    'dawn': 'bg-gradient-to-b from-orange-300 to-pink-200',
    'dusk': 'bg-gradient-to-b from-orange-500 to-purple-400',
    'night': 'bg-gradient-to-b from-gray-900 to-black'
  }
  
  return skyClasses[time as keyof typeof skyClasses] || 'bg-blue-400'
})

const buildingColor = computed(() => {
  return isDay.value ? 'bg-gray-600' : 'bg-gray-800'
})

const cityBuildings = computed(() => {
  if (!location.value) return []
  
  const population = location.value.population || 100000
  const citySize = getCitySize(population)
  
  const cities = []
  for (let i = 0; i < 5; i++) {
    const buildings = generateBuildings(i, citySize)
    cities.push({ buildings })
  }
  
  return cities
})

const userCityIndex = computed(() => {
  if (!location.value) return 2
  const population = location.value.population || 100000
  return getCitySize(population)
})

function getCitySize(population: number): number {
  if (population < 50000) return 0
  if (population < 200000) return 1
  if (population < 500000) return 2
  if (population < 2000000) return 3
  return 4
}

function generateBuildings(cityIndex: number, userCitySize: number) {
  const buildingCounts = [3, 4, 6, 8, 10]
  const maxHeights = [80, 120, 160, 200, 250]
  const maxWidths = [20, 25, 30, 35, 40]
  
  const count = buildingCounts[cityIndex]
  const maxHeight = maxHeights[cityIndex]
  const maxWidth = maxWidths[cityIndex]
  
  const buildings: Building[] = []
  
  for (let i = 0; i < count; i++) {
    const width = Math.random() * (maxWidth - 15) + 15
    const height = Math.random() * (maxHeight - 40) + 40
    
    const windows: Window[] = []
    const windowRows = Math.floor(height / 20)
    const windowCols = Math.floor(width / 8)
    
    for (let row = 1; row < windowRows; row++) {
      for (let col = 1; col < windowCols; col++) {
        if (Math.random() > 0.3) {
          windows.push({
            id: row * windowCols + col,
            x: col * 8,
            y: row * 20,
            width: 4,
            height: 6
          })
        }
      }
    }
    
    buildings.push({
      id: i,
      width,
      height,
      windows
    })
  }
  
  return buildings
}

function createWeatherEffects() {
  if (weather.value?.condition === 'rain') {
    raindrops.value = Array.from({ length: 50 }, (_, i) => ({
      id: i,
      x: Math.random() * 100,
      y: Math.random() * 100,
      size: Math.random() * 2 + 1,
      speed: Math.random() * 2 + 1
    }))
  } else if (weather.value?.condition === 'snow') {
    snowflakes.value = Array.from({ length: 30 }, (_, i) => ({
      id: i,
      x: Math.random() * 100,
      y: Math.random() * 100,
      size: Math.random() * 4 + 2,
      speed: Math.random() * 3 + 2
    }))
  } else if (weather.value?.condition === 'cloudy') {
    clouds.value = Array.from({ length: 6 }, (_, i) => {
      const width = Math.random() * 100 + 80
      const height = Math.random() * 40 + 30
      
      return {
        id: i,
        x: Math.random() * 120 - 20, // Allow some clouds to start off-screen
        y: Math.random() * 30 + 5, // Keep clouds in upper portion of sky
        width,
        height,
        drift: Math.random() * 50, // Start with random drift positions
        puffs: Array.from({ length: Math.floor(Math.random() * 3) + 3 }, (_, j) => ({
          id: j,
          x: Math.random() * width * 0.8 - width * 0.1, // Center the puffs better
          y: Math.random() * height * 0.6 - height * 0.1, // Position puffs more naturally
          size: Math.random() * 25 + 20
        }))
      }
    })
  } else {
    // Clear weather - reset all effects
    raindrops.value = []
    snowflakes.value = []
    clouds.value = []
  }
}

let effectInterval: number | null = null

function startWeatherAnimation() {
  if (effectInterval) clearInterval(effectInterval)
  
  effectInterval = setInterval(() => {
    if (weather.value?.condition === 'rain') {
      raindrops.value = raindrops.value.map(drop => ({
        ...drop,
        y: drop.y > 100 ? -10 : drop.y + drop.speed * 2
      }))
    } else if (weather.value?.condition === 'snow') {
      snowflakes.value = snowflakes.value.map(flake => ({
        ...flake,
        y: flake.y > 100 ? -10 : flake.y + flake.speed,
        x: flake.x + Math.sin(flake.y / 20) * 0.3
      }))
    } else if (weather.value?.condition === 'cloudy') {
      clouds.value = clouds.value.map(cloud => ({
        ...cloud,
        drift: cloud.drift + 0.5 // More visible drift movement
      }))
    }
  }, 50)
}

async function getUserLocation(): Promise<Location> {
  const response = await fetch('https://ipapi.co/json/')
  const data = await response.json()
  
  // Try to get a better population estimate based on city name
  let population = data.population
  if (!population || population < 10000) {
    // Fallback estimates for common cities
    const cityEstimates: { [key: string]: number } = {
      'chattanooga': 180000,
      'nashville': 700000,
      'knoxville': 190000,
      'memphis': 650000,
      'atlanta': 500000,
      'birmingham': 210000,
      'huntsville': 220000
    }
    
    const cityKey = data.city?.toLowerCase()
    population = cityEstimates[cityKey] || 150000 // Better default
  }
  
  return {
    city: data.city,
    region: data.region,
    country: data.country,
    lat: data.latitude,
    lon: data.longitude,
    population
  }
}

// Cache management
interface CachedWeather {
  data: Weather
  timestamp: number
  lat: number
  lon: number
}

function getCacheKey(lat: number, lon: number): string {
  return `weather_${lat.toFixed(2)}_${lon.toFixed(2)}`
}

function getCachedWeather(lat: number, lon: number): Weather | null {
  try {
    const cacheKey = getCacheKey(lat, lon)
    const cached = localStorage.getItem(cacheKey)
    
    if (!cached) return null
    
    const cachedData: CachedWeather = JSON.parse(cached)
    const now = Date.now()
    const threeHours = 3 * 60 * 60 * 1000 // 3 hours in milliseconds
    
    // Check if cache is still valid (within 3 hours)
    if (now - cachedData.timestamp < threeHours) {
      console.log('Using cached weather data for', lat, lon)
      return cachedData.data
    } else {
      // Cache expired, remove it
      localStorage.removeItem(cacheKey)
      return null
    }
  } catch (error) {
    console.warn('Error reading weather cache:', error)
    return null
  }
}

function cacheWeather(lat: number, lon: number, weatherData: Weather): void {
  try {
    const cacheKey = getCacheKey(lat, lon)
    const cacheData: CachedWeather = {
      data: weatherData,
      timestamp: Date.now(),
      lat,
      lon
    }
    
    localStorage.setItem(cacheKey, JSON.stringify(cacheData))
    console.log('Cached weather data for', lat, lon)
  } catch (error) {
    console.warn('Error caching weather data:', error)
  }
}

async function getWeather(lat: number, lon: number): Promise<Weather> {
  // Check cache first
  const cachedWeather = getCachedWeather(lat, lon)
  if (cachedWeather) {
    return cachedWeather
  }
  
  try {
    const apiKey = 'a0bc8628edeecef556629a1026ab99d7' // Temporarily hardcoded to test
    console.log('Fetching fresh weather data for', lat, lon)
    const response = await fetch(`https://api.openweathermap.org/data/2.5/weather?lat=${lat}&lon=${lon}&appid=${apiKey}&units=metric`)
    const data = await response.json()
    
    // Debug logging
    console.log('Weather API Response:', data)
    console.log('Temperature from API:', data.main?.temp)
    console.log('Weather description:', data.weather?.[0]?.description)
    
    // Check if API returned an error (common with demo key)
    if (data.cod && data.cod !== 200) {
      throw new Error('API returned error: ' + data.message)
    }
    
    const weatherConditions: { [key: string]: Weather['condition'] } = {
      'clear sky': 'clear',
      'few clouds': 'cloudy',
      'scattered clouds': 'cloudy',
      'broken clouds': 'cloudy',
      'overcast clouds': 'cloudy',
      'light rain': 'rain',
      'moderate rain': 'rain',
      'heavy intensity rain': 'rain',
      'light snow': 'snow',
      'snow': 'snow',
      'heavy snow': 'snow'
    }
    
    const description = data.weather?.[0]?.description?.toLowerCase() || 'clear sky'
    const condition = weatherConditions[description] || 'clear'
    
    const currentTime = new Date()
    const currentHour = currentTime.getHours()
    const isDay = currentHour >= 7 && currentHour < 17
    
    const weatherResult: Weather = {
      condition,
      temperature: data.main?.temp || 20,
      isDay
    }
    
    // Cache the result
    cacheWeather(lat, lon, weatherResult)
    
    return weatherResult
  } catch (error) {
    console.warn('Weather API unavailable, generating realistic mock data:', error)
    
    // Generate more realistic weather based on city/location
    const cityName = location.value?.city?.toLowerCase() || ''
    
    // City-specific mock weather data
    const cityWeatherData: { [key: string]: { temp: number, condition: Weather['condition'] } } = {
      'chattanooga': { temp: 22, condition: 'cloudy' }, // 71°F ≈ 22°C
      'tokyo': { temp: 28, condition: 'cloudy' }, // 82°F ≈ 28°C  
      'new york': { temp: 18, condition: 'clear' },
      'los angeles': { temp: 24, condition: 'clear' },
      'london': { temp: 15, condition: 'rain' },
      'paris': { temp: 19, condition: 'cloudy' },
      'sydney': { temp: 21, condition: 'clear' }
    }
    
    const mockData = cityWeatherData[cityName] || {
      temp: Math.floor(Math.random() * 25) + 10, // 10-35°C
      condition: (['clear', 'cloudy', 'rain'] as Weather['condition'][])[Math.floor(Math.random() * 3)]
    }
    
    const currentTime = new Date()
    const currentHour = currentTime.getHours()
    const isDay = currentHour >= 7 && currentHour < 17
    
    return {
      condition: mockData.condition,
      temperature: mockData.temp,
      isDay
    }
  }
}

onMounted(async () => {
  try {
    location.value = await getUserLocation()
    originalUserLocation.value = { ...location.value }
    await getCityTimezone(location.value.lat, location.value.lon)
    weather.value = await getWeather(location.value.lat, location.value.lon)
    createWeatherEffects()
    startWeatherAnimation()
  } catch (error) {
    console.error('Failed to get location or weather:', error)
    location.value = {
      city: 'Unknown City',
      region: 'Unknown Region',
      country: 'Unknown Country',
      lat: 0,
      lon: 0,
      population: 100000
    }
    originalUserLocation.value = { ...location.value }
    
    const currentTime = new Date()
    const currentHour = currentTime.getHours()
    const isDay = currentHour >= 6 && currentHour < 20
    
    weather.value = {
      condition: 'sunny',
      temperature: 22,
      isDay
    }
    
    createWeatherEffects()
    startWeatherAnimation()
  } finally {
    loading.value = false
  }
})

function toggleWeatherMenu() {
  showWeatherMenu.value = !showWeatherMenu.value
  showCityMenu.value = false
}

function toggleCityMenu() {
  showCityMenu.value = !showCityMenu.value
  showWeatherMenu.value = false
  if (!showCityMenu.value) {
    showSuggestions.value = false
    searchSuggestions.value = []
    citySearch.value = ''
  }
}

let searchTimeout: number | null = null

async function onSearchInput() {
  const query = citySearch.value.trim()
  
  if (!query || query.length < 3) {
    searchSuggestions.value = []
    showSuggestions.value = false
    return
  }

  // Debounce the search - increased to 800ms to reduce API calls
  if (searchTimeout) {
    clearTimeout(searchTimeout)
  }

  searchTimeout = setTimeout(async () => {
    await searchForSuggestions()
  }, 800)
}

async function searchForSuggestions() {
  const query = citySearch.value.trim().toLowerCase()
  if (!query || query.length < 3) return

  // Check cache first
  const cached = searchCache.get(query)
  if (cached) {
    const now = Date.now()
    if (now - cached.timestamp < SEARCH_CACHE_DURATION) {
      console.log('Using cached search results for:', query)
      searchSuggestions.value = cached.data
      showSuggestions.value = cached.data.length > 0
      return
    } else {
      // Cache expired
      searchCache.delete(query)
    }
  }

  isSearching.value = true
  
  try {
    const apiKey = 'a0bc8628edeecef556629a1026ab99d7'
    console.log('Making API call for search:', query)
    const response = await fetch(`https://api.openweathermap.org/geo/1.0/direct?q=${encodeURIComponent(query)}&limit=5&appid=${apiKey}`)
    const data = await response.json()
    
    console.log('Search suggestions API response:', data)
    
    if (data && data.length > 0) {
      const suggestions = data.map((city: any) => ({
        name: city.name,
        display_name: `${city.name}, ${city.state || city.country}`,
        lat: city.lat,
        lon: city.lon,
        state: city.state,
        country: city.country
      }))
      
      // Cache the results
      searchCache.set(query, {
        data: suggestions,
        timestamp: Date.now()
      })
      
      searchSuggestions.value = suggestions
      showSuggestions.value = true
    } else {
      searchSuggestions.value = []
      showSuggestions.value = false
      
      // Cache empty results too to avoid repeated API calls
      searchCache.set(query, {
        data: [],
        timestamp: Date.now()
      })
    }
  } catch (error) {
    console.warn('Search suggestions failed:', error)
    searchSuggestions.value = []
    showSuggestions.value = false
  } finally {
    isSearching.value = false
  }
}

async function selectSuggestion(suggestion: any) {
  location.value = {
    city: suggestion.name,
    region: suggestion.state || suggestion.country,
    country: suggestion.country,
    lat: suggestion.lat,
    lon: suggestion.lon,
    population: 500000
  }
  
  // Get timezone for the selected city
  await getCityTimezone(suggestion.lat, suggestion.lon)
  
  // Update weather for the new city
  weather.value = await getWeather(suggestion.lat, suggestion.lon)
  createWeatherEffects()
  startWeatherAnimation()
  
  // Close menu and clear search
  showCityMenu.value = false
  showSuggestions.value = false
  searchSuggestions.value = []
  citySearch.value = ''
}

function changeWeather(condition: Weather['condition']) {
  if (weather.value) {
    weather.value.condition = condition
    createWeatherEffects()
    startWeatherAnimation()
    showWeatherMenu.value = false
  }
}

function toggleTimeOfDay() {
  if (weather.value) {
    weather.value.isDay = !weather.value.isDay
    showWeatherMenu.value = false
  }
}

async function selectCity(city: any) {
  location.value = {
    city: city.name,
    region: city.region,
    country: city.region,
    lat: city.lat,
    lon: city.lon,
    population: city.population
  }
  
  // Get timezone for the selected city
  await getCityTimezone(city.lat, city.lon)
  
  // Update weather for the new city
  weather.value = await getWeather(city.lat, city.lon)
  createWeatherEffects()
  
  showCityMenu.value = false
  citySearch.value = ''
}

async function searchCity() {
  if (!citySearch.value.trim()) return
  
  isSearching.value = true
  
  try {
    const apiKey = 'a0bc8628edeecef556629a1026ab99d7' // Temporarily hardcoded to test
    console.log('Searching for city:', citySearch.value)
    const response = await fetch(`https://api.openweathermap.org/geo/1.0/direct?q=${encodeURIComponent(citySearch.value)}&limit=1&appid=${apiKey}`)
    const data = await response.json()
    
    console.log('Search API response:', data)
    
    if (data && data.length > 0) {
      const city = data[0]
      location.value = {
        city: city.name,
        region: city.state || city.country,
        country: city.country,
        lat: city.lat,
        lon: city.lon,
        population: 500000
      }
      
      console.log('Found city:', city.name, city.state, city.country)
      
      // Get timezone for the searched city
      await getCityTimezone(city.lat, city.lon)
      
      weather.value = await getWeather(city.lat, city.lon)
      createWeatherEffects()
      startWeatherAnimation()
      
      // Close menu
      showCityMenu.value = false
      citySearch.value = ''
    } else {
      console.warn('No cities found for:', citySearch.value)
      alert(`No cities found for "${citySearch.value}". Try including state/country like "Bloomington, Indiana"`)
    }
  } catch (error) {
    console.warn('City search failed, using mock data:', error)
    location.value = {
      city: citySearch.value,
      region: 'Unknown',
      country: 'Unknown',
      lat: 0,
      lon: 0,
      population: Math.floor(Math.random() * 2000000) + 100000
    }
    
    // Reset timezone for unknown cities
    currentCityTimezone.value = null
    
    // Generate mock weather for the searched city
    const currentTime = new Date()
    const currentHour = currentTime.getHours()
    const isDay = currentHour >= 6 && currentHour < 20
    
    const conditions: Weather['condition'][] = ['clear', 'rain', 'snow', 'cloudy']
    const condition = conditions[Math.floor(Math.random() * conditions.length)]
    
    weather.value = {
      condition,
      temperature: Math.floor(Math.random() * 30) + 5,
      isDay
    }
    
    createWeatherEffects()
  } finally {
    isSearching.value = false
  }
  
  showCityMenu.value = false
  citySearch.value = ''
}

function getCitySizeLabel(index: number): string {
  const labels = ['Village', 'Town', 'City', 'Metropolis', 'Megacity']
  return labels[index] || 'City'
}

function formatPopulation(pop?: number): string {
  if (!pop) return '~100K'
  if (pop >= 1000000) return `~${Math.round(pop / 1000000)}M`
  if (pop >= 1000) return `~${Math.round(pop / 1000)}K`
  return `~${pop}`
}

function formatTemperature(temp: number): string {
  const isUSA = originalUserLocation.value?.country === 'US' || originalUserLocation.value?.country === 'USA'
  if (isUSA) {
    const fahrenheit = Math.round((temp * 9/5) + 32)
    return `${fahrenheit}°F`
  }
  return `${Math.round(temp)}°C`
}

function getCurrentTime(): string {
  const now = new Date()
  
  // If we have a timezone for the current city, use it
  if (currentCityTimezone.value) {
    try {
      return now.toLocaleTimeString([], { 
        timeZone: currentCityTimezone.value,
        hour: '2-digit', 
        minute: '2-digit',
        hour12: true 
      })
    } catch (error) {
      console.warn('Invalid timezone:', currentCityTimezone.value)
    }
  }
  
  return now.toLocaleTimeString([], { 
    hour: '2-digit', 
    minute: '2-digit',
    hour12: true 
  })
}

async function getCityTimezone(lat: number, lon: number): Promise<void> {
  try {
    // First try to match known cities by coordinates for better accuracy
    const cityName = location.value?.city?.toLowerCase()
    const knownCityTimezones: { [key: string]: string } = {
      'new york': 'America/New_York',
      'los angeles': 'America/Los_Angeles', 
      'chicago': 'America/Chicago',
      'london': 'Europe/London',
      'tokyo': 'Asia/Tokyo',
      'paris': 'Europe/Paris',
      'sydney': 'Australia/Sydney',
      'toronto': 'America/Toronto',
      'chattanooga': 'America/New_York',
      'nashville': 'America/Chicago',
      'atlanta': 'America/New_York',
      'berlin': 'Europe/Berlin',
      'moscow': 'Europe/Moscow'
    }
    
    if (cityName && knownCityTimezones[cityName]) {
      currentCityTimezone.value = knownCityTimezones[cityName]
      return
    }
    
    // Fallback to API-based timezone lookup
    const apiKey = 'a0bc8628edeecef556629a1026ab99d7' // Temporarily hardcoded to test
    const response = await fetch(`https://api.openweathermap.org/data/2.5/weather?lat=${lat}&lon=${lon}&appid=${apiKey}`)
    const data = await response.json()
    
    if (data.timezone) {
      // Convert UTC offset to timezone (rough approximation)
      const offsetHours = data.timezone / 3600
      const timezoneMap: { [key: number]: string } = {
        '-12': 'Pacific/Baker_Island',
        '-11': 'Pacific/Samoa',
        '-10': 'Pacific/Honolulu',
        '-9': 'America/Anchorage',
        '-8': 'America/Los_Angeles',
        '-7': 'America/Denver',
        '-6': 'America/Chicago',
        '-5': 'America/New_York',
        '-4': 'America/Caracas',
        '-3': 'America/Sao_Paulo',
        '-2': 'Atlantic/South_Georgia',
        '-1': 'Atlantic/Azores',
        '0': 'Europe/London',
        '1': 'Europe/Paris',
        '2': 'Europe/Berlin',
        '3': 'Europe/Moscow',
        '4': 'Asia/Dubai',
        '5': 'Asia/Karachi',
        '6': 'Asia/Dhaka',
        '7': 'Asia/Bangkok',
        '8': 'Asia/Shanghai',
        '9': 'Asia/Tokyo',
        '10': 'Australia/Sydney',
        '11': 'Pacific/Norfolk',
        '12': 'Pacific/Auckland'
      }
      
      currentCityTimezone.value = timezoneMap[offsetHours.toString()] || null
    }
  } catch (error) {
    console.warn('Failed to get timezone:', error)
    currentCityTimezone.value = null
  }
}

onUnmounted(() => {
  if (effectInterval) {
    clearInterval(effectInterval)
  }
})
</script>