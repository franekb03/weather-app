<script>
export default {
  name: 'App',

  data() {
    return {
      query: '',
      city: null,
      current: null,
      daily: [],
      loading: false,
      error: null
    }
  },

  methods: {
    async searchCity() {
      const name = this.query.trim()
      if (!name) return

      try {
        this.error = null
        this.loading = true
        const url =
          'https://geocoding-api.open-meteo.com/v1/search' +
          `?name=${encodeURIComponent(name)}&count=1`
        const response = await fetch(url)
        const data = await response.json()

        if (!data.results) {
          console.warn(`No city found for "${name}"`)
          return
        }

        const { name: cityName, country, latitude, longitude } = data.results[0]
        this.city = { name: cityName, country, latitude, longitude }
        await this.loadWeather()
      } catch (e) {
        console.error('Search failed:', e)
        this.error = e
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
    </form>

    <template v-if="loading">
      <p>Loading...</p>
    </template>

    <template v-else-if="error">
      <p>Failed: {{error}}</p>
    </template>

    <template v-else>
      <section class="current">
        <h2>{{ city.name }}, {{ city.country }}</h2>
        <p>{{ current.temperature_2m }}°C</p>
        <p>Weather code: {{ current.weather_code }}</p>
        <p>Wind {{ current.wind_speed_10m }} km/h · Humidity {{ current.relative_humidity_2m }}%</p>
      </section>

      <ul class="forecast">
        <li v-for="day in daily" :key="day.date">
          <span>{{ day.date }}</span>
          <span>Code {{ day.code }}</span>
          <span>{{ day.max }}° / {{ day.min }}°</span>
        </li>
      </ul>
    </template>
  </main>
</template>