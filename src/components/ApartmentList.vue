<script setup>
import { ref } from 'vue'

// Eure 2 Apartments aus Milestone 1:
const apartments = ref([
  {
    id: 1,
    title: 'Helles WG-Zimmer am Alex',
    price: 380.0,
    city: 'Berlin',
    district: 'Mitte'
  },
  {
    id: 2,
    title: 'Ruhiges Zimmer fuer Studenten',
    price: 420.0,
    city: 'Berlin',
    district: 'Karlshorst'
  }
])
</script>

<template>
  <div class="apartment-container">
    <h2>Verfügbare Apartments</h2>
    <div class="apartment-list">
      <!-- Rendering mit v-for -->
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
</style>