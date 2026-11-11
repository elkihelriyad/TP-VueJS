<template>
  <div class="app">
    <header class="topbar">
      <h1>Météo en direct</h1>
      <button class="unit-toggle" @click="basculerUnite" v-if="ville">
        Afficher en {{ unite === 'metric' ? '°F' : '°C' }}
      </button>
    </header>

    <div class="search">
      <input
        v-model="query"
        @input="onInputRecherche"
        type="text"
        placeholder="Rechercher une ville... (ex: Safi, Paris, Tokyo)"
      />
      <p class="hint" v-if="chargementSuggestions">Recherche...</p>
      <p class="msg-error" v-if="erreurSuggestions">{{ erreurSuggestions }}</p>

      <ul class="suggestions" v-if="suggestions.length">
        <li v-for="s in suggestions" :key="s.lat + '-' + s.lon" @click="choisirVille(s)">
          {{ s.name }}<span v-if="s.state">, {{ s.state }}</span> ({{ s.country }})
        </li>
      </ul>
    </div>

    <div class="historique" v-if="historique.length">
      <span>Récent :</span>
      <button v-for="h in historique" :key="h.lat + '-' + h.lon" @click="choisirVille(h)">
        {{ h.name }}
      </button>
    </div>

    <div class="meteo" v-if="ville">
      <p class="loading" v-if="chargementMeteo">Chargement de la météo...</p>
      <p class="msg-error" v-else-if="erreurMeteo">{{ erreurMeteo }}</p>

      <div v-else-if="meteoActuelle" class="courante">
        <h2>{{ ville.name }}, {{ ville.country }}</h2>
        <div class="temp-actuelle">
          <img :src="iconeUrl(meteoActuelle.weather[0].icon)" :alt="meteoActuelle.weather[0].description" />
          <span class="valeur">{{ Math.round(meteoActuelle.main.temp) }}°</span>
          <span class="description">{{ meteoActuelle.weather[0].description }}</span>
        </div>
        <p class="details">
          Ressenti {{ Math.round(meteoActuelle.main.feels_like) }}° · Humidité {{ meteoActuelle.main.humidity }}%
        </p>
      </div>

      <div class="previsions" v-if="previsionsParJour.length">
        <div class="jour" v-for="j in previsionsParJour" :key="j.date">
          <span class="date">{{ j.date }}</span>
          <img :src="iconeUrl(j.icone)" :alt="j.description" />
          <span class="temps">{{ Math.round(j.min) }}° / {{ Math.round(j.max) }}°</span>
        </div>
      </div>
    </div>

    <p class="empty" v-if="!ville">Cherchez une ville pour voir sa météo.</p>
  </div>
</template>

<script>
// La cle est lue depuis .env (jamais committee -- voir .env.example et le sujet).
const API_KEY = import.meta.env.VITE_OWM_API_KEY
const GEO_URL = 'https://api.openweathermap.org/geo/1.0/direct'
const WEATHER_URL = 'https://api.openweathermap.org/data/2.5/weather'
const FORECAST_URL = 'https://api.openweathermap.org/data/2.5/forecast'

export default {
  name: 'App',
  data() {
    return {
      query: '',
      debounceTimer: null,
      suggestions: [],
      chargementSuggestions: false,
      erreurSuggestions: null,

      ville: null, 
      historique: [],

      meteoActuelle: null,
      previsions: [], 
      chargementMeteo: false,
      erreurMeteo: null,

      unite: 'metric'
    }
  },
  computed: {
    previsionsParJour() {
      const jours = {}

      for (const p of this.previsions) {
        const date = p.dt_txt.split(' ')[0]
        const heure = parseInt(p.dt_txt.split(' ')[1].slice(0, 2), 10)
        const ecartMidi = Math.abs(heure - 12)

        if (!jours[date]) {
          jours[date] = {
            date,
            min: p.main.temp_min,
            max: p.main.temp_max,
            icone: p.weather[0].icon,
            description: p.weather[0].description,
            ecartMidi
          }
        } else {
          const j = jours[date]
          j.min = Math.min(j.min, p.main.temp_min)
          j.max = Math.max(j.max, p.main.temp_max)
          // tranche plus proche de midi : on garde son icone/description
          if (ecartMidi < j.ecartMidi) {
            j.icone = p.weather[0].icon
            j.description = p.weather[0].description
            j.ecartMidi = ecartMidi
          }
        }
      }

      return Object.values(jours)
        .sort((a, b) => a.date.localeCompare(b.date))
        .map(({ date, min, max, icone, description }) => ({ date, min, max, icone, description }))
    }
  },
  methods: {
    iconeUrl(code) {
      return `https://openweathermap.org/img/wn/${code}@2x.png`
    },
    onInputRecherche() {
      clearTimeout(this.debounceTimer)

      if (!this.query.trim()) {
        this.suggestions = []
        this.erreurSuggestions = null
        this.chargementSuggestions = false
        return
      }

      this.debounceTimer = setTimeout(() => {
        this.rechercherVilles()
      }, 400)
    },
    async rechercherVilles() {
      this.chargementSuggestions = true
      this.erreurSuggestions = null

      try {
        const reponse = await fetch(
          `${GEO_URL}?q=${encodeURIComponent(this.query)}&limit=5&appid=${API_KEY}`
        )

        if (reponse.status === 401) {
          throw new Error("Clé API invalide, absente ou pas encore activée. Vérifiez votre fichier .env.")
        }
        if (reponse.status === 429) {
          throw new Error('Quota de requêtes dépassé. Veuillez réessayer plus tard.')
        }
        if (!reponse.ok) {
          throw new Error(`Erreur du serveur (code ${reponse.status}).`)
        }

        const donnees = await reponse.json()

        if (!Array.isArray(donnees)) {
          throw new Error('Réponse inattendue du serveur.')
        }
        if (donnees.length === 0) {
          this.suggestions = []
          this.erreurSuggestions = 'Aucune ville trouvée pour cette recherche.'
          return
        }

        this.suggestions = donnees
      } catch (e) {
        this.suggestions = []
        if (e instanceof TypeError) {
          this.erreurSuggestions = 'Erreur réseau : impossible de contacter le serveur. Vérifiez votre connexion.'
        } else {
          this.erreurSuggestions = e.message
        }
      } finally {
        this.chargementSuggestions = false
      }
    },
    choisirVille(ville) {
      clearTimeout(this.debounceTimer)
      this.suggestions = []
      this.query = ''
      this.chargementSuggestions = false
      this.erreurSuggestions = null

      this.ville = ville

      const autres = this.historique.filter(
        (h) => !(h.lat === ville.lat && h.lon === ville.lon)
      )
      this.historique = [ville, ...autres].slice(0, 5)
      this.sauvegarderHistorique()

      this.chargerMeteo()
    },
    async chargerMeteo() {
      if (!this.ville) return

      const { lat, lon } = this.ville
      this.chargementMeteo = true
      this.erreurMeteo = null

      try {
        const [repMeteo, repPrevisions] = await Promise.all([
          fetch(`${WEATHER_URL}?lat=${lat}&lon=${lon}&appid=${API_KEY}&units=${this.unite}&lang=fr`),
          fetch(`${FORECAST_URL}?lat=${lat}&lon=${lon}&appid=${API_KEY}&units=${this.unite}&lang=fr`)
        ])

        for (const rep of [repMeteo, repPrevisions]) {
          if (rep.status === 401) {
            throw new Error("Clé API invalide, absente ou pas encore activée. Vérifiez votre fichier .env.")
          }
          if (rep.status === 429) {
            throw new Error('Quota de requêtes dépassé. Veuillez réessayer plus tard.')
          }
          if (!rep.ok) {
            throw new Error(`Erreur du serveur (code ${rep.status}).`)
          }
        }

        const [meteo, donneesPrevisions] = await Promise.all([
          repMeteo.json(),
          repPrevisions.json()
        ])

        if (!meteo || !meteo.main || !Array.isArray(meteo.weather) || !meteo.weather.length ||
            !donneesPrevisions || !Array.isArray(donneesPrevisions.list)) {
          throw new Error('Réponse inattendue du serveur.')
        }

        this.meteoActuelle = meteo
        this.previsions = donneesPrevisions.list
      } catch (e) {
        this.meteoActuelle = null
        this.previsions = []
        if (e instanceof TypeError) {
          this.erreurMeteo = 'Erreur réseau : impossible de contacter le serveur. Vérifiez votre connexion.'
        } else {
          this.erreurMeteo = e.message
        }
      } finally {
        this.chargementMeteo = false
      }
    },
    basculerUnite() {
      this.unite = this.unite === 'metric' ? 'imperial' : 'metric'
      this.chargerMeteo()
    },

    sauvegarderHistorique() {
      try {
        localStorage.setItem('historiqueVilles', JSON.stringify(this.historique))
      } catch (e) {
      }
    }
  },
  created() {
    try {
      const sauvegarde = JSON.parse(localStorage.getItem('historiqueVilles'))
      if (Array.isArray(sauvegarde)) {
        this.historique = sauvegarde.slice(0, 5)
      }
    } catch (e) {
      this.historique = []
    }
  }
}
</script>
