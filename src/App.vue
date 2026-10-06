<script>
const WEATHER_CODES = {
  0: ['Clear sky', '☀️'],
  1: ['Mostly clear', '🌤️'],
  2: ['Partly cloudy', '⛅'],
  3: ['Overcast', '☁️'],
  45: ['Fog', '🌫️'],
  48: ['Fog', '🌫️'],
  51: ['Light drizzle', '🌦️'],
  53: ['Drizzle', '🌦️'],
  55: ['Heavy drizzle', '🌧️'],
  61: ['Light rain', '🌦️'],
  63: ['Rain', '🌧️'],
  65: ['Heavy rain', '🌧️'],
  71: ['Light snow', '🌨️'],
  73: ['Snow', '❄️'],
  75: ['Heavy snow', '❄️'],
  80: ['Rain showers', '🌦️'],
  95: ['Thunderstorm', '⛈️']
}

export default {
  name: 'App',

  data() {
    return {
      query: '',
      city: null, // { name, country, latitude, longitude }
      current: null,
      daily: [], // [{ date, code, max, min }]
      unit: 'C',
      loading: false,
      error: '',
      recent: []
    }
  },

  computed: {
    currentTemp() {
      return this.current ? this.convert(this.current.temperature_2m) : ''
    },

    temperatureClass() {
      if (!this.current) return ''
      const t = this.current.temperature_2m
      if (t <= 0) return 'freezing'
      if (t < 15) return 'cool'
      if (t < 25) return 'mild'
      return 'hot'
    }
  },

  watch: {
    // Fires whenever searchCity stores a new city object
    city() {
      this.loadWeather()
    }
  },

  mounted() {
    this.query = 'Warsaw'
    this.searchCity()
  },

  methods: {
    convert(celsius) {
      const value = this.unit === 'C' ? celsius : (celsius * 9) / 5 + 32
      return Math.round(value)
    },

    toggleUnit() {
      this.unit = this.unit === 'C' ? 'F' : 'C'
    },

    describe(code) {
      const [label, icon] = WEATHER_CODES[code] || ['Unknown', '❔']
      return { label, icon }
    },

    formatDay(date) {
      return new Date(date).toLocaleDateString('en-GB', { weekday: 'short' })
    },

    selectRecent(name) {
      this.query = name
      this.searchCity()
    },

    // Finds the city only. The watcher loads the forecast.
    async searchCity() {
      const name = this.query.trim()
      if (!name) return

      this.loading = true
      this.error = ''

      try {
        const url =
          'https://geocoding-api.open-meteo.com/v1/search' +
          `?name=${encodeURIComponent(name)}&count=1`
        const response = await fetch(url)
        if (!response.ok) throw new Error('City search failed')
        const data = await response.json()

        if (!data.results) throw new Error(`No city found for "${name}"`)

        const { name: cityName, country, latitude, longitude } = data.results[0]

        this.recent = [cityName, ...this.recent.filter(c => c !== cityName)].slice(0, 5)
        this.city = { name: cityName, country, latitude, longitude }
      } catch (e) {
        this.error = e.message
        this.loading = false
      }
    },

    async loadWeather() {
      this.loading = true
      this.error = ''

      try {
        const { latitude, longitude } = this.city
        const url =
          'https://api.open-meteo.com/v1/forecast' +
          `?latitude=${latitude}&longitude=${longitude}` +
          '&current=temperature_2m,relative_humidity_2m,wind_speed_10m,weather_code' +
          '&daily=weather_code,temperature_2m_max,temperature_2m_min' +
          '&timezone=auto'

        const response = await fetch(url)
        if (!response.ok) throw new Error('Weather service returned an error')
        const data = await response.json()

        this.current = data.current
        this.daily = data.daily.time.map((date, i) => ({
          date,
          code: data.daily.weather_code[i],
          max: data.daily.temperature_2m_max[i],
          min: data.daily.temperature_2m_min[i]
        }))
      } catch (e) {
        this.error = e.message
      } finally {
        this.loading = false
      }
    }
  }
}
</script>

<template>
  <main class="app">
    <h1>Weather</h1>

    <form class="search" @submit.prevent="searchCity">
      <input v-model="query" placeholder="Search city…" />
      <button type="submit">Go</button>
      <button type="button" @click="toggleUnit">°{{ unit }}</button>
    </form>

    <div class="chips">
      <button
        v-for="name in recent"
        :key="name"
        type="button"
        @click="selectRecent(name)"
      >
        {{ name }}
      </button>
    </div>

    <p v-if="loading">Loading…</p>
    <p v-else-if="error" class="error">{{ error }}</p>

    <template v-else-if="current">
      <section class="current" :class="temperatureClass">
        <h2>{{ city.name }}, {{ city.country }}</h2>
        <p class="big">{{ describe(current.weather_code).icon }} {{ currentTemp }}°{{ unit }}</p>
        <p>{{ describe(current.weather_code).label }}</p>
        <p>Wind {{ current.wind_speed_10m }} km/h · Humidity {{ current.relative_humidity_2m }}%</p>
      </section>

      <ul class="forecast">
        <li v-for="day in daily" :key="day.date">
          <span>{{ formatDay(day.date) }}</span>
          <span>{{ describe(day.code).icon }}</span>
          <span>{{ convert(day.max) }}° / {{ convert(day.min) }}°</span>
        </li>
      </ul>
    </template>
  </main>
</template>

<style scoped>
.app {
  max-width: 420px;
  margin: 2rem auto;
  padding: 0 1rem;
  font-family: system-ui, sans-serif;
}

.search {
  display: flex;
  gap: 0.5rem;
}

.search input {
  flex: 1;
  padding: 0.5rem;
}

.chips {
  display: flex;
  flex-wrap: wrap;
  gap: 0.4rem;
  margin: 0.6rem 0;
}

.current {
  margin: 1rem 0;
  padding: 1rem;
  border-radius: 12px;
  color: #111;
  transition: background-color 0.4s;
}

.big {
  margin: 0.2rem 0;
  font-size: 2.2rem;
}

.freezing { background: #cfe8ff; }
.cool     { background: #dff3e4; }
.mild     { background: #fff3c4; }
.hot      { background: #ffd2c2; }

.forecast {
  list-style: none;
  padding: 0;
}

.forecast li {
  display: grid;
  grid-template-columns: 4rem 3rem 1fr;
  gap: 0.5rem;
  padding: 0.4rem 0;
  border-bottom: 1px solid #ddd;
}

.forecast li span:last-child {
  text-align: right;
}

.error {
  color: #b00020;
}
</style>