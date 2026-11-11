<template>
  <h1>Explorateur de pays</h1>

  <p class="loading" v-show="loading">Chargement des pays...</p>
  <p class="error" v-if="error">{{ error }}</p>

  <div class="toolbar">
    <input v-model="recherche" placeholder="Rechercher un pays..." />
  </div>

  <div class="grid">
    <CountryCard v-for="country in paysAffiches" :key="country.cca3" :country="country" />
  </div>
</template>

<script>
import CountryCard from '../components/CountryCard.vue'


const API_URL = 'https://raw.githubusercontent.com/DovAzencot/ApiCountries/main/data.json'

function parserListe(texte) {
  const t = texte.trim().replace(/,$/, '')
  return JSON.parse(t.startsWith('[') ? t : '[' + t + ']')
}

function adapterPays(p) {
  return {
    name: { common: p.name, official: p.name },
    capital: p.capital ? [p.capital] : [],
    region: p.region,
    population: p.population,
    flags: { svg: p.flags && (p.flags.svg || p.flags.png) },
    cca3: p.alpha3Code
  }
}

export default {
  name: 'CountryListView',
  components: { CountryCard },
  data() {
    return {
      countries: [],
      loading: true,
      error: null,
      recherche: ''
    }
  },
  computed: {
    paysAffiches() {
      const terme = this.recherche.trim().toLowerCase()
      if (!terme) return this.countries
      return this.countries.filter(country =>
        country.name.common.toLowerCase().includes(terme)
      )
    }
  },
  methods: {
    async chargerPays() {
      this.loading = true
      this.error = null
      try {
        const response = await fetch(API_URL)
        if (!response.ok) {
          throw new Error(`Erreur HTTP ${response.status}`)
        }
        const data = parserListe(await response.text())
        this.countries = data.map(adapterPays)
      } catch (e) {
        this.error = 'Impossible de charger la liste des pays. Vérifiez votre connexion et réessayez. (' + e.message + ')'
      } finally {
        this.loading = false
      }
    }
  },
  mounted() {
    this.chargerPays()
  }
}
</script>
