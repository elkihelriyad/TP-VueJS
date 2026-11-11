<template>
  <p class="loading" v-show="loading">Chargement...</p>
  <p class="error" v-if="error">{{ error }}</p>

  <div v-if="country">
    <img :src="country.flags.svg" :alt="`Drapeau ${country.name.common}`" style="width: 200px;" />
    <h1>{{ country.name.common }}</h1>
    <p>Nom officiel : {{ country.name.official }}</p>
    <p>Capitale : {{ country.capital ? country.capital[0] : '—' }}</p>
    <p>Région : {{ country.region }}</p>
    <p>Population : {{ country.population.toLocaleString() }}</p>
  </div>

  <button @click="$router.back()">Retour</button>
</template>

<script>
const API_URL = 'https://raw.githubusercontent.com/DovAzencot/ApiCountries/main/data.json'

// Le fichier est une suite d'objets séparés par des virgules (sans [ ]) :
// on l'entoure de crochets avant de le parser
function parserListe(texte) {
  const t = texte.trim().replace(/,$/, '')
  return JSON.parse(t.startsWith('[') ? t : '[' + t + ']')
}

// Adapte un pays d'apicountries.com au format utilisé par le template
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
  name: 'CountryDetailView',
  data() {
    return {
      country: null,
      loading: true,
      error: null
    }
  },
  methods: {
    // TODO 3 : même principe que chargerPays() dans CountryListView, mais
    // sur l'API (recherche du pays par son code cca3) où <code>
    // vient de this.$route.params.code. La réponse peut être un objet ou un
    // tableau d'un seul élément selon les cas -- gérez les deux.
    async chargerPays() {
      this.loading = true
      this.error = null
      this.country = null
      try {
        const code = this.$route.params.code
        const response = await fetch(API_URL)
        if (!response.ok) {
          throw new Error(`Erreur HTTP ${response.status}`)
        }
        const data = parserListe(await response.text())
        // Le résultat peut être un objet unique ou un tableau : on gère les deux
        const trouve = data.find(p => p.alpha3Code === String(code).toUpperCase())
        const pays = Array.isArray(trouve) ? trouve[0] : trouve
        if (!pays) {
          throw new Error(`Pays introuvable (code « ${code} »).`)
        }
        this.country = adapterPays(pays)
      } catch (e) {
        this.error = 'Impossible de charger ce pays : ' + e.message
      } finally {
        this.loading = false
      }
    }
  },
  watch: {
    // Bonus : recharger si le code change sans recréer le composant
    '$route.params.code'(nouveauCode) {
      if (nouveauCode) this.chargerPays()
    }
  },
  mounted() {
    this.chargerPays()
  }
}
</script>
