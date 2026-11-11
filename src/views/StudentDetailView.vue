<template>
  <div v-if="student">
    <h1>{{ student.firstName }} {{ student.lastName }}</h1>

    <p>Filière : <input v-model="student.filiere" /></p>
    <p>Année : <input v-model.number="student.year" type="number" min="1" /></p>
    <p>Email : <input v-model="student.email" type="email" /></p>
  </div>
  <div v-else>
    <p>Étudiant introuvable.</p>
  </div>

  <button @click="$router.push('/')">Retour</button>
</template>

<script>
import { store } from '../store/students.js'

export default {
  name: 'StudentDetailView',
  computed: {
    // Renvoie l'objet original du store (pas une copie) pour que v-model le modifie directement
    student() {
      const id = Number(this.$route.params.id)
      return store.students.find(s => s.id === id)
    }
  }
}
</script>
