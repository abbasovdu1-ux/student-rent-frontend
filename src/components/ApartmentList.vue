<script setup>
import { ref, onMounted } from 'vue'

// Am Anfang ist die Liste leer, bis die Daten vom Backend eintreffen
const apartments = ref([])

// Funktion, die die Daten per GET abruft
const fetchApartments = async () => {
  try {
    // Eure Backend-URL auf Render:
    const response = await fetch('https://student-rent-project.onrender.com/api/apartments')
    const data = await response.json()
    apartments.value = data
  } catch (error) {
    console.error('Fehler beim Laden der Apartments:', error)
  }
}

// Wird automatisch beim Laden der Seite aufgerufen
onMounted(() => {
  fetchApartments()
})
</script>

<template>
  <div class="apartment-container">
    <h2>Verfügbare Apartments</h2>

    <!-- Zeigt Text an, solange noch keine Daten geladen wurden -->
    <div v-if="apartments.length === 0" class="loading">
      Lade Apartments...
    </div>

    <!-- Gewohnter v-for-Block -->
    <div v-else class="apartment-list">
      <div v-for="apartment in apartments" :key="apartment.id" class="apartment-card">
        <h3>{{ apartment.title }}</h3>
        <p><strong>Preis:</strong> {{ apartment.price }} €</p>
        <p><strong>Lage:</strong> {{ apartment.city }} - {{ apartment.district }}</p>
      </div>
    </div>
  </div>
</template>

<style scoped>
.apartment-container {
  max-width: 600px;
  margin: 0 auto;
  padding: 20px;
}
.apartment-card {
  border: 1px solid #444;
  border-radius: 8px;
  padding: 16px;
  margin-bottom: 16px;
  background-color: #1a1a1a;
  color: #fff;
  text-align: left;
}
.loading {
  color: #888;
  padding: 20px;
}
</style>