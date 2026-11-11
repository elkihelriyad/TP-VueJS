<template>
  <h1>Annuaire des étudiants</h1>

  <form @submit.prevent="ajouterEtudiant">
    <input v-model="newStudent.firstName" placeholder="Prénom" />
    <input v-model="newStudent.lastName" placeholder="Nom" />
    <input v-model="newStudent.filiere" placeholder="Filière" />
    <input v-model.number="newStudent.year" type="number" min="1" placeholder="Année" />
    <input v-model="newStudent.email" type="email" placeholder="Email" />
    <button type="submit">Ajouter</button>
  </form>

  <div class="toolbar">
    <input v-model="recherche" placeholder="Rechercher par nom ou prénom" />
    <select v-model="tri">
      <option value="lastName">Trier par nom</option>
      <option value="filiere">Trier par filière</option>
    </select>
  </div>

  <StudentCard v-for="student in etudiantsAffiches" :key="student.id" :student="student" />
</template>

<script>
import StudentCard from '../components/StudentCard.vue'
import { store } from '../store/students.js'

export default {
  name: 'StudentListView',
  components: { StudentCard },
  data() {
    return {
      store,
      recherche: '',
      tri: 'lastName',
      newStudent: { firstName: '', lastName: '', filiere: '', year: 1, email: '' }
    }
  },
  computed: {
    etudiantsAffiches() {
      const texte = this.recherche.trim().toLowerCase()
      // filter() crée déjà un nouveau tableau : store.students n'est jamais muté par le tri
      const filtres = store.students.filter(s =>
        s.firstName.toLowerCase().includes(texte) ||
        s.lastName.toLowerCase().includes(texte)
      )
      const cle = this.tri
      return [...filtres].sort((a, b) => a[cle].localeCompare(b[cle]))
    }
  },
  methods: {
    ajouterEtudiant() {
      if (!this.newStudent.firstName.trim() || !this.newStudent.lastName.trim()) {
        return
      }
      const id = store.students.reduce((max, s) => Math.max(max, s.id), 0) + 1
      store.students.push({ ...this.newStudent, id })
      this.newStudent = { firstName: '', lastName: '', filiere: '', year: 1, email: '' }
    }
  }
}
</script>
