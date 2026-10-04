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
      city: null,
      current: null, // from the API, always in °C
      daily: [],
      unit: 'C',
      loading: false,
      error: ''
    }
  },

  mounted() {
    this.query = 'Warsaw'
    this.searchCity()
  },

  computed: {
    currentTemp() {
      return this.current ? this.convert(this.current.temperature_2m) : ''
    }
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
        this.city = { name: cityName, country, latitude, longitude }
        await this.loadWeather()
      } catch (e) {
        this.error = e.message
      } finally {
        this.loading = false
      }
    },

    async loadWeather() {
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
    }
  }
}
</script>

<template>
  <main>
    <h1>Weather</h1>

    <form @submit.prevent="searchCity">
      <input v-model="query" placeholder="Search city…" />
      <button type="submit">Go</button>
      <button type="button" @click="toggleUnit">°{{ unit }}</button>
    </form>

    <p v-if="loading">Loading…</p>
    <p v-else-if="error" class="error">{{ error }}</p>

    <template v-else-if="current">
      <section class="current">
        <h2>{{ city.name }}, {{ city.country }}</h2>
        <p>{{ describe(current.weather_code).icon }} {{ currentTemp }}°{{ unit }}</p>
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