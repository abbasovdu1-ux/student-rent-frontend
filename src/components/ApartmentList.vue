<script setup>
import { ref, onMounted } from 'vue'

// Am Anfang ist die Liste leer, bis die Daten vom Backend eintreffen
const apartments = ref([])

// Variablen fuer die Eingabefelder im Formular
const title = ref('')
const price = ref('')
const city = ref('')
const district = ref('')

// Eure Backend-URL auf Render:
const API_URL = 'https://student-rent-project.onrender.com/api/apartments'

// Funktion, die die Daten per GET abruft
const fetchApartments = async () => {
  try {
    const response = await fetch(API_URL)
    const data = await response.json()

    // Daten in der Variable speichern
    apartments.value = data
  } catch (error) {
    console.error('Fehler beim Laden der Apartments:', error)
  }
}

// Funktion, die ein neues Apartment per POST an das Backend schickt
const createApartment = async () => {
  try {
    // Neues Apartment-Objekt mit den Werten aus dem Formular erstellen
    const newApartment = {
      title: title.value,
      price: parseFloat(price.value),
      city: city.value,
      district: district.value
    }

    // Daten per POST an das Backend senden
    const response = await fetch(API_URL, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(newApartment)
    })

    // Wenn das Speichern erfolgreich war
    if (response.ok) {
      // Eingabefelder nach dem Absenden wieder leeren
      title.value = ''
      price.value = ''
      city.value = ''
      district.value = ''

      // Liste neu laden, damit das neue Apartment direkt angezeigt wird
      await fetchApartments()
    }
  } catch (error) {
    console.error('Fehler beim Erstellen des Apartments:', error)
  }
}

// Wird automatisch beim Laden der Seite aufgerufen
onMounted(() => {
  fetchApartments()
})
</script>

<template>
  <div class="apartment-container">
    <h2>Neues Apartment hinzufügen</h2>

    <!-- Formular zum Hinzufügen eines neuen Apartments -->
    <form @submit.prevent="createApartment" class="apartment-form">
      <input v-model="title" type="text" placeholder="Titel (z.B. WG-Zimmer am Alex)" required />
      <input v-model="price" type="number" step="0.01" placeholder="Preis in € (z.B. 400)" required />
      <input v-model="city" type="text" placeholder="Stadt (z.B. Berlin)" required />
      <input v-model="district" type="text" placeholder="Bezirk (z.B. Mitte)" required />
      <button type="submit">Apartment speichern</button>
    </form>

    <hr class="divider" />

    <h2>Verfügbare Apartments</h2>

    <!-- Zeigt Text an, solange noch keine Daten geladen wurden oder keine vorhanden sind -->
    <div v-if="apartments.length === 0" class="loading">
      Noch keine Apartments vorhanden. Füge oben das erste hinzu!
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

/* Styling für das Eingabeformular */
.apartment-form {
  display: flex;
  flex-direction: column;
  gap: 12px;
  background-color: #1a1a1a;
  padding: 16px;
  border-radius: 8px;
  border: 1px solid #444;
  margin-bottom: 24px;
}

.apartment-form input {
  padding: 10px;
  border-radius: 4px;
  border: 1px solid #555;
  background-color: #2a2a2a;
  color: #fff;
}

.apartment-form button {
  padding: 10px;
  background-color: #42b883;
  color: #fff;
  border: none;
  border-radius: 4px;
  font-weight: bold;
  cursor: pointer;
}

.apartment-form button:hover {
  background-color: #33a06f;
}

.divider {
  border: 0;
  border-top: 1px solid #444;
  margin: 24px 0;
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