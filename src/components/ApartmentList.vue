<script setup>
import { ref, computed, onMounted } from 'vue'

// --- STATE ---
const apartments = ref([])
const loading = ref(true)
const lang = ref('de')
const isDarkMode = ref(true)
const favorites = ref(new Set())

// Filter & Standort ("In deiner Nähe")
const activeFeedMode = ref('all') // 'all' oder 'nearby'
const userCity = ref(null)
const locationLoading = ref(false)
const filterMinPrice = ref('')
const filterMaxPrice = ref('')
const filterCity = ref('')
const filterType = ref('ALL')

// Modals
const detailItem = ref(null)
const selectedLandlord = ref(null)
const filterModal = ref(false)
const addModal = ref(false)
const settingsModal = ref(false)
const isPublishing = ref(false)
const formError = ref('')

// Formular für neues Inserat (entspricht 1:1 den Feldern in Apartment.java)
const newTitle = ref('')
const newPriceRaw = ref('')
const pricePeriod = ref('Monat')
const newType = ref('Langzeit')
const newCity = ref('')
const newDistrict = ref('')
const newSpace = ref('')
const newPhone = ref('')
const newDesc = ref('')
const newLandlordName = ref('')
const newImageUrl = ref('')

const API_URL = 'https://student-rent-project.onrender.com/api/apartments'

// --- ÜBERSETZUNGEN (DE / RU) ---
const TRANSLATIONS = {
  de: {
    filters: 'Filter',
    nearby: 'In deiner Nähe',
    allFeeds: 'Alle',
    longTerm: 'Langzeit',
    shortTerm: 'Kurzzeit',
    call: 'Anrufen',
    whatsapp: 'WhatsApp',
    showDetails: 'Details anzeigen',
    close: 'Schließen',
    delete: 'Löschen',
    apply: 'Filter anwenden',
    reset: 'Zurücksetzen',
    publish: 'Inserat veröffentlichen',
    publishing: 'Wird gespeichert...',
    newListing: 'Inserieren',
    parameters: 'Angaben',
    available: 'Verfügbar ab',
    mandatoryError: 'Bitte Titel, Preis, Stadt und deinen Namen ausfüllen!',
    verified: 'Konto aktiv',
    memberSince: 'Mitglied seit',
    perMonth: '/ Monat',
    perDay: '/ Tag',
    rentalPeriod: 'Mietdauer',
    settings: 'Einstellungen',
    language: 'Sprache',
    theme: 'Design',
    darkMode: 'Dunkel',
    lightMode: 'Hell',
    description: 'Beschreibung',
    otherListings: 'Alle Inserate von',
    noResults: 'Noch keine passenden Wohnungen vorhanden.',
    loading: 'Lade Wohnungen aus PostgreSQL...'
  },
  ru: {
    filters: 'Фильтры',
    nearby: 'Рядом',
    allFeeds: 'Все',
    longTerm: 'Долгосрок',
    shortTerm: 'Посуточно',
    call: 'Позвонить',
    whatsapp: 'WhatsApp',
    showDetails: 'Подробнее',
    close: 'Закрыть',
    delete: 'Удалить',
    apply: 'Применить',
    reset: 'Сбросить',
    publish: 'Опубликовать',
    publishing: 'Сохранение...',
    newListing: 'Создать',
    parameters: 'Параметры',
    available: 'Доступно',
    mandatoryError: 'Заполните название, цену, город и ваше имя!',
    verified: 'Аккаунт активен',
    memberSince: 'В сервисе с',
    perMonth: '/ мес',
    perDay: '/ день',
    rentalPeriod: 'Срок аренды',
    settings: 'Настройки',
    language: 'Язык',
    theme: 'Тема оформления',
    darkMode: 'Тёмная',
    lightMode: 'Светлая',
    description: 'Описание',
    otherListings: 'Все объявления от',
    noResults: 'Подходящих квартир пока нет.',
    loading: 'Загрузка из PostgreSQL...'
  }
}

const t = computed(() => TRANSLATIONS[lang.value])

// Echte Zeitberechnung aus dem Server-Timestamp (Instant createdAt)
const getRelativeTimeString = (isoTimestamp) => {
  if (!isoTimestamp) return lang.value === 'de' ? 'Neu' : 'Новое'
  const createdMs = new Date(isoTimestamp).getTime()
  const diffInMinutes = Math.max(0, Math.floor((Date.now() - createdMs) / (1000 * 60)))
  const diffInHours = Math.floor(diffInMinutes / 60)
  const diffInDays = Math.floor(diffInHours / 24)

  if (lang.value === 'de') {
    if (diffInMinutes < 1) return 'Gerade eben'
    if (diffInMinutes < 60) return `vor ${diffInMinutes} Min.`
    if (diffInHours < 24) return `vor ${diffInHours} Std.`
    return `vor ${diffInDays} Tagen`
  } else {
    if (diffInMinutes < 1) return 'Только что'
    if (diffInMinutes < 60) return `${diffInMinutes} мин. назад`
    if (diffInHours < 24) return `${diffInHours} ч. назад`
    return `${diffInDays} дн. назад`
  }
}

const DEFAULT_FALLBACK_IMAGE = 'https://images.unsplash.com/photo-1522708323590-d24dbb6b0267?w=1000'

// --- 1. GET: ECHTE DATEN VOM BACKEND LADEN ---
const fetchListings = async () => {
  loading.value = true
  try {
    const res = await fetch(API_URL)
    const data = await res.json()
    // Neueste Inserate (höchste ID) zuerst anzeigen
    apartments.value = [...data].sort((a, b) => (b.id || 0) - (a.id || 0))
  } catch (err) {
    console.error('Fehler beim Laden vom Backend:', err)
  } finally {
    loading.value = false
  }
}

// --- STANDORT-ABFRAGE ("IN DEINER NÄHE") ---
const toggleNearby = () => {
  if (activeFeedMode.value === 'nearby') {
    activeFeedMode.value = 'all'
    return
  }
  activeFeedMode.value = 'nearby'
  if (userCity.value) return

  if (!navigator.geolocation) {
    alert('Standort wird von diesem Browser nicht unterstützt.')
    activeFeedMode.value = 'all'
    return
  }

  locationLoading.value = true
  navigator.geolocation.getCurrentPosition(
    async (pos) => {
      try {
        const { latitude, longitude } = pos.coords
        const geoRes = await fetch(
          `https://nominatim.openstreetmap.org/reverse?format=json&lat=${latitude}&lon=${longitude}`
        )
        const geoData = await geoRes.json()
        userCity.value =
          geoData.address?.city ||
          geoData.address?.town ||
          geoData.address?.state ||
          'Berlin'
      } catch {
        userCity.value = 'Berlin'
      } finally {
        locationLoading.value = false
      }
    },
    () => {
      locationLoading.value = false
      alert('Standortzugriff wurde verweigert.')
      activeFeedMode.value = 'all'
    }
  )
}

// --- GEFILTERTE LISTE ---
const filteredApartments = computed(() => {
  return apartments.value.filter((apt) => {
    if (activeFeedMode.value === 'nearby' && userCity.value) {
      if (apt.city?.toLowerCase() !== userCity.value.toLowerCase()) return false
    }
    if (filterType.value !== 'ALL' && (apt.type || 'Langzeit') !== filterType.value) return false
    if (filterCity.value && !apt.city?.toLowerCase().includes(filterCity.value.toLowerCase())) return false
    if (filterMinPrice.value && apt.price < parseFloat(filterMinPrice.value)) return false
    if (filterMaxPrice.value && apt.price > parseFloat(filterMaxPrice.value)) return false
    return true
  })
})

// Alle echten Inserate eines Vermieters aus der Datenbank
const landlordListings = computed(() => {
  if (!selectedLandlord.value) return []
  return apartments.value.filter(
    (a) => (a.landlordName || 'Student_Vermieter') === selectedLandlord.value.name
  )
})

const openLandlordProfile = (apt) => {
  selectedLandlord.value = {
    name: apt.landlordName || 'Student_Vermieter',
    memberSince: apt.memberSince || '2026'
  }
}

// --- 2. POST: NEUES INSERAT DAUERHAFT IN POSTGRESQL SPEICHERN ---
const handleAddListing = async () => {
  if (!newTitle.value.trim() || !newPriceRaw.value || !newCity.value.trim() || !newLandlordName.value.trim()) {
    formError.value = t.value.mandatoryError
    return
  }

  isPublishing.value = true
  formError.value = ''

  const payload = {
    title: newTitle.value.trim(),
    price: parseFloat(newPriceRaw.value),
    priceSuffix: pricePeriod.value === 'Monat' ? t.value.perMonth : t.value.perDay,
    type: newType.value,
    city: newCity.value.trim(),
    district: newDistrict.value.trim() || 'Zentrum',
    space: newSpace.value ? `${newSpace.value} m²` : null,
    phone: newPhone.value.trim() || null,
    description: newDesc.value.trim() || null,
    imageUrl: newImageUrl.value.trim() || null,
    landlordName: newLandlordName.value.trim(),
    memberSince: new Date().getFullYear().toString()
  }

  try {
    const res = await fetch(API_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    })

    if (res.ok) {
      newTitle.value = ''
      newPriceRaw.value = ''
      newCity.value = ''
      newDistrict.value = ''
      newSpace.value = ''
      newPhone.value = ''
      newDesc.value = ''
      newLandlordName.value = ''
      newImageUrl.value = ''
      addModal.value = false
      await fetchListings()
    } else {
      formError.value = 'Serverfehler beim Speichern.'
    }
  } catch {
    formError.value = 'Verbindungsfehler zum Backend.'
  } finally {
    isPublishing.value = false
  }
}

// --- 3. DELETE: ECHTES LÖSCHEN IN POSTGRESQL ---
const handleDelete = async (id) => {
  if (!confirm('Dieses Inserat endgültig aus der Datenbank löschen?')) return
  try {
    const res = await fetch(`${API_URL}/${id}`, { method: 'DELETE' })
    if (res.ok) {
      apartments.value = apartments.value.filter((item) => item.id !== id)
      if (detailItem.value?.id === id) detailItem.value = null
    }
  } catch (err) {
    console.error('Fehler beim Löschen:', err)
  }
}

const toggleFavorite = (id) => {
  if (favorites.value.has(id)) favorites.value.delete(id)
  else favorites.value.add(id)
}

const resetFilters = () => {
  filterMinPrice.value = ''
  filterMaxPrice.value = ''
  filterCity.value = ''
  filterType.value = 'ALL'
  activeFeedMode.value = 'all'
  filterModal.value = false
}

onMounted(() => {
  fetchListings()
})
</script>

<template>
  <div class="app-root">
    <!-- Schwebende Top-Bar -->
    <header class="top-bar">
      <div class="top-left-group">
        <button class="pill-btn" @click="filterModal = true">
          ▽ {{ t.filters }}
        </button>
        <button
          :class="['pill-btn', { 'pill-active': activeFeedMode === 'nearby' }]"
          @click="toggleNearby"
        >
          📍 {{ locationLoading ? '...' : userCity && activeFeedMode === 'nearby' ? userCity : t.nearby }}
        </button>
      </div>

      <div class="top-right-group">
        <button class="circle-icon-btn" @click="settingsModal = true" title="Einstellungen">
          ⚙
        </button>
        <button class="insert-btn" @click="addModal = true">
          ＋ {{ t.newListing }}
        </button>
      </div>
    </header>

    <!-- Vertikaler Reels Snap-Container -->
    <main class="reels-feed">
      <div v-if="loading" class="empty-screen">
        <p>{{ t.loading }}</p>
      </div>

      <div v-else-if="filteredApartments.length === 0" class="empty-screen">
        <p>{{ t.noResults }}</p>
        <button class="insert-btn" @click="addModal = true">＋ {{ t.newListing }}</button>
      </div>

      <section
        v-else
        v-for="item in filteredApartments"
        :key="item.id"
        class="reel-card"
      >
        <!-- Hintergrundbild (echte imageUrl oder Standard-Bild) -->
        <img
          :src="item.imageUrl || DEFAULT_FALLBACK_IMAGE"
          class="reel-image"
          alt="Apartment"
        />

        <!-- Dunkler Verlauf unten -->
        <div class="overlay-gradient"></div>

        <!-- Rechte Floating-Buttons -->
        <div class="right-floating-bar">
          <button class="float-circle" @click="handleDelete(item.id)" :title="t.delete">
            🗑
          </button>

          <button class="float-circle" @click="toggleFavorite(item.id)">
            <span :style="{ color: favorites.has(item.id) ? '#FF3B30' : '#fff', fontSize: '22px' }">
              {{ favorites.has(item.id) ? '♥' : '♡' }}
            </span>
          </button>
        </div>

        <!-- Unterer Content-Bereich -->
        <div class="bottom-overlay">
          <!-- Klickbares Vermieter-Konto -->
          <div class="landlord-chip" @click="openLandlordProfile(item)">
            <div class="mini-avatar">{{ (item.landlordName || 'S').charAt(0).toUpperCase() }}</div>
            <span>{{ item.landlordName || 'Student_Vermieter' }}</span>
            <span class="since-tag">· seit {{ item.memberSince || '2026' }}</span>
          </div>

          <!-- Badges Row -->
          <div class="badge-row">
            <span class="badge-orange">
              {{ (item.type || 'Langzeit') === 'Langzeit' ? t.longTerm : t.shortTerm }}
            </span>
            <span class="badge-glass">
              {{ getRelativeTimeString(item.createdAt) }}
            </span>
            <span v-if="item.space" class="badge-glass">
              {{ item.space }}
            </span>
          </div>

          <div class="price-text">
            {{ item.price }} € <span class="price-suffix">{{ item.priceSuffix || t.perMonth }}</span>
          </div>

          <h2 class="reel-title">{{ item.title }}</h2>
          <p class="reel-location">{{ item.city }}, {{ item.district }}</p>

          <button class="details-btn" @click="detailItem = item">
            <span>{{ t.showDetails }}</span>
            <span>↑</span>
          </button>
        </div>
      </section>
    </main>

    <!-- MODAL 1: DETAILS ANZEIGEN -->
    <div v-if="detailItem" class="fullscreen-modal" :class="{ 'light-theme': !isDarkMode }">
      <div class="modal-inner">
        <div class="detail-header">
          <button class="back-link" @click="detailItem = null">
            &lt; {{ detailItem.city }}
          </button>
          <button class="close-link" @click="detailItem = null">
            {{ t.close }}
          </button>
        </div>

        <div class="detail-scroll">
          <div class="detail-gallery">
            <img :src="detailItem.imageUrl || DEFAULT_FALLBACK_IMAGE" class="detail-img" />
          </div>

          <div class="detail-body">
            <div class="badge-row">
              <span class="badge-orange">
                {{ (detailItem.type || 'Langzeit') === 'Langzeit' ? t.longTerm : t.shortTerm }}
              </span>
              <span class="badge-neutral">
                {{ getRelativeTimeString(detailItem.createdAt) }}
              </span>
            </div>

            <div class="detail-price">
              {{ detailItem.price }} € <span class="price-suffix">{{ detailItem.priceSuffix || t.perMonth }}</span>
            </div>
            <h1 class="detail-title">{{ detailItem.title }}</h1>
            <p class="detail-loc">{{ detailItem.city }}, {{ detailItem.district }}</p>

            <div v-if="detailItem.space" class="amenities-row">
              <span class="amenity-chip">{{ detailItem.space }}</span>
            </div>

            <hr class="divider" />

            <!-- Vermieter-Profil Karte -->
            <div class="landlord-box" @click="openLandlordProfile(detailItem)">
              <div class="avatar-circle">{{ (detailItem.landlordName || 'S').charAt(0).toUpperCase() }}</div>
              <div>
                <div class="landlord-title">{{ detailItem.landlordName || 'Student_Vermieter' }}</div>
                <div class="verified-green">✓ {{ t.verified }}</div>
                <div class="member-sub">{{ t.memberSince }} {{ detailItem.memberSince || '2026' }}</div>
              </div>
            </div>

            <hr class="divider" />
            <h3>{{ t.description }}</h3>
            <p class="desc-text">
              {{ detailItem.description || 'Keine zusätzliche Beschreibung angegeben.' }}
            </p>
          </div>
        </div>

        <!-- Feste Kontakt-Leiste unten (wenn Telefonnummer angegeben) -->
        <div v-if="detailItem.phone" class="detail-bottom-bar">
          <a
            :href="`https://wa.me/${detailItem.phone.replace(/[^0-9]/g, '')}`"
            target="_blank"
            class="wa-btn"
          >
            💬 {{ t.whatsapp }}
          </a>
          <a :href="`tel:${detailItem.phone}`" class="call-btn">
            📞 {{ t.call }}
          </a>
        </div>
      </div>
    </div>

    <!-- MODAL 2: VERMIETER-KONTO / PROFIL -->
    <div v-if="selectedLandlord" class="sheet-backdrop" @click.self="selectedLandlord = null">
      <div class="bottom-sheet" :class="{ 'light-theme': !isDarkMode }">
        <div class="drag-bar"></div>
        <div class="profile-top">
          <div class="avatar-circle large">{{ selectedLandlord.name.charAt(0).toUpperCase() }}</div>
          <h2>{{ selectedLandlord.name }}</h2>
          <p class="verified-green">✓ {{ t.verified }}</p>
          <p class="member-sub">{{ t.memberSince }} {{ selectedLandlord.memberSince }}</p>
        </div>

        <h4>{{ t.otherListings }} {{ selectedLandlord.name }} ({{ landlordListings.length }})</h4>
        <div class="mini-list">
          <div
            v-for="apt in landlordListings"
            :key="apt.id"
            class="mini-apt-row"
            @click="selectedLandlord = null; detailItem = apt"
          >
            <div>
              <strong>{{ apt.title }}</strong>
              <div class="member-sub">{{ apt.city }}, {{ apt.district }} · {{ getRelativeTimeString(apt.createdAt) }}</div>
            </div>
            <span class="mini-price">{{ apt.price }} €</span>
          </div>
        </div>
      </div>
    </div>

    <!-- MODAL 3: FILTER -->
    <div v-if="filterModal" class="sheet-backdrop" @click.self="filterModal = false">
      <div class="bottom-sheet" :class="{ 'light-theme': !isDarkMode }">
        <div class="drag-bar"></div>
        <h3>{{ t.filters }}</h3>

        <label class="field-label">{{ t.rentalPeriod }}</label>
        <div class="segment-row">
          <button :class="['seg-btn', { active: filterType === 'ALL' }]" @click="filterType = 'ALL'">{{ t.allFeeds }}</button>
          <button :class="['seg-btn', { active: filterType === 'Langzeit' }]" @click="filterType = 'Langzeit'">{{ t.longTerm }}</button>
          <button :class="['seg-btn', { active: filterType === 'Kurzzeit' }]" @click="filterType = 'Kurzzeit'">{{ t.shortTerm }}</button>
        </div>

        <label class="field-label">Preis (€)</label>
        <div class="two-cols">
          <input v-model="filterMinPrice" type="number" placeholder="von €" class="sheet-input" />
          <input v-model="filterMaxPrice" type="number" placeholder="bis €" class="sheet-input" />
        </div>

        <label class="field-label">Stadt</label>
        <input v-model="filterCity" placeholder="z. B. Berlin" class="sheet-input" />

        <div class="two-cols" style="margin-top: 12px;">
          <button class="secondary-action" @click="resetFilters">{{ t.reset }}</button>
          <button class="primary-action" @click="filterModal = false">{{ t.apply }}</button>
        </div>
      </div>
    </div>

    <!-- MODAL 4: EINSTELLUNGEN -->
    <div v-if="settingsModal" class="sheet-backdrop" @click.self="settingsModal = false">
      <div class="bottom-sheet" :class="{ 'light-theme': !isDarkMode }">
        <div class="drag-bar"></div>
        <h3>⚙ {{ t.settings }}</h3>

        <label class="field-label">{{ t.theme }}</label>
        <div class="segment-row">
          <button :class="['seg-btn', { active: isDarkMode }]" @click="isDarkMode = true">🌙 {{ t.darkMode }}</button>
          <button :class="['seg-btn', { active: !isDarkMode }]" @click="isDarkMode = false">☀ {{ t.lightMode }}</button>
        </div>

        <label class="field-label" style="margin-top: 14px;">{{ t.language }}</label>
        <div class="segment-row">
          <button :class="['seg-btn', { active: lang === 'de' }]" @click="lang = 'de'">🇩🇪 DE</button>
          <button :class="['seg-btn', { active: lang === 'ru' }]" @click="lang = 'ru'">🇷🇺 RU</button>
        </div>
      </div>
    </div>

    <!-- MODAL 5: NEUES INSERAT ERSTELLEN -->
    <div v-if="addModal" class="sheet-backdrop" @click.self="!isPublishing && (addModal = false)">
      <div class="bottom-sheet tall-sheet" :class="{ 'light-theme': !isDarkMode }">
        <div class="drag-bar"></div>
        <h3>{{ t.newListing }}</h3>

        <p v-if="formError" class="error-banner">{{ formError }}</p>

        <div class="segment-row">
          <button :class="['seg-btn orange', { active: newType === 'Langzeit' }]" @click="newType = 'Langzeit'">{{ t.longTerm }}</button>
          <button :class="['seg-btn orange', { active: newType === 'Kurzzeit' }]" @click="newType = 'Kurzzeit'">{{ t.shortTerm }}</button>
        </div>

        <input v-model="newLandlordName" placeholder="Dein Name / Account-Name *" class="sheet-input" />
        <input v-model="newTitle" placeholder="Titel der Anzeige *" class="sheet-input" />

        <div class="two-cols">
          <input v-model="newPriceRaw" type="number" placeholder="Preis € *" class="sheet-input" />
          <div class="segment-row" style="margin: 0;">
            <button :class="['seg-btn', { active: pricePeriod === 'Monat' }]" @click="pricePeriod = 'Monat'">{{ t.perMonth }}</button>
            <button :class="['seg-btn', { active: pricePeriod === 'Tag' }]" @click="pricePeriod = 'Tag'">{{ t.perDay }}</button>
          </div>
        </div>

        <div class="two-cols">
          <input v-model="newCity" placeholder="Stadt *" class="sheet-input" />
          <input v-model="newDistrict" placeholder="Bezirk" class="sheet-input" />
        </div>

        <div class="two-cols">
          <input v-model="newSpace" type="number" placeholder="Fläche (m²)" class="sheet-input" />
          <input v-model="newPhone" placeholder="Telefon / WhatsApp" class="sheet-input" />
        </div>

        <input v-model="newImageUrl" placeholder="Bild-URL (optional, z. B. https://...)" class="sheet-input" />
        <textarea v-model="newDesc" placeholder="Beschreibung..." rows="3" class="sheet-input"></textarea>

        <button class="primary-action full-w" :disabled="isPublishing" @click="handleAddListing">
          {{ isPublishing ? t.publishing : t.publish }}
        </button>
      </div>
    </div>
  </div>
</template>

<style>
html, body, #app {
  margin: 0 !important;
  padding: 0 !important;
  width: 100% !important;
  height: 100% !important;
  background-color: #000000 !important;
  overflow: hidden;
  font-family: -apple-system, BlinkMacSystemFont, 'SF Pro Display', 'Segoe UI', Roboto, sans-serif;
}
</style>

<style scoped>
.app-root {
  position: relative;
  width: 100vw;
  height: 100vh;
  background: #000;
  display: flex;
  justify-content: center;
}

.top-bar {
  position: fixed;
  top: 22px;
  width: 100%;
  max-width: 560px;
  padding: 0 16px;
  box-sizing: border-box;
  z-index: 100;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.top-left-group, .top-right-group {
  display: flex;
  align-items: center;
  gap: 10px;
}

.pill-btn {
  background: rgba(0, 0, 0, 0.55);
  backdrop-filter: blur(10px);
  color: #fff;
  border: 1px solid rgba(255, 255, 255, 0.15);
  padding: 9px 15px;
  border-radius: 20px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
}

.pill-active {
  background: #FF9500;
  border-color: #FF9500;
  font-weight: 600;
}

.circle-icon-btn {
  background: rgba(0, 0, 0, 0.55);
  backdrop-filter: blur(10px);
  color: #fff;
  border: 1px solid rgba(255, 255, 255, 0.15);
  width: 38px;
  height: 38px;
  border-radius: 50%;
  font-size: 18px;
  cursor: pointer;
}

.insert-btn {
  background: #007AFF;
  color: #fff;
  border: none;
  padding: 10px 15px;
  border-radius: 18px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
}

.reels-feed {
  width: 100%;
  max-width: 560px;
  height: 100vh;
  overflow-y: scroll;
  scroll-snap-type: y mandatory;
  scrollbar-width: none;
}
.reels-feed::-webkit-scrollbar {
  display: none;
}

.reel-card {
  position: relative;
  width: 100%;
  height: 100vh;
  scroll-snap-align: start;
  scroll-snap-stop: always;
  overflow: hidden;
  background: #111;
}

.reel-image {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.overlay-gradient {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  height: 48%;
  background: linear-gradient(transparent, rgba(0, 0, 0, 0.92));
  pointer-events: none;
}

.right-floating-bar {
  position: absolute;
  right: 16px;
  top: 46%;
  z-index: 30;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 14px;
}

.float-circle {
  width: 44px;
  height: 44px;
  border-radius: 50%;
  background: rgba(0, 0, 0, 0.45);
  backdrop-filter: blur(6px);
  border: 1px solid rgba(255, 255, 255, 0.18);
  color: #fff;
  font-size: 18px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
}

.bottom-overlay {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  padding: 0 18px 26px;
  z-index: 25;
  color: #fff;
}

.landlord-chip {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  background: rgba(255, 255, 255, 0.16);
  backdrop-filter: blur(8px);
  padding: 5px 12px 5px 6px;
  border-radius: 99px;
  font-size: 13px;
  font-weight: 600;
  margin-bottom: 10px;
  cursor: pointer;
}

.mini-avatar {
  width: 24px;
  height: 24px;
  border-radius: 50%;
  background: #007AFF;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 12px;
}

.since-tag {
  color: #cbd5e1;
  font-weight: 400;
}

.badge-row {
  display: flex;
  gap: 8px;
  margin-bottom: 10px;
}

.badge-orange {
  background: #FF9500;
  color: #fff;
  font-size: 12px;
  font-weight: 600;
  padding: 4px 9px;
  border-radius: 6px;
}

.badge-glass {
  background: rgba(255, 255, 255, 0.2);
  color: #fff;
  font-size: 12px;
  font-weight: 500;
  padding: 4px 9px;
  border-radius: 6px;
}

.badge-neutral {
  background: rgba(142, 142, 147, 0.25);
  font-size: 12px;
  padding: 4px 9px;
  border-radius: 6px;
}

.price-text {
  font-size: 28px;
  font-weight: 700;
  margin-bottom: 4px;
}

.price-suffix {
  font-size: 17px;
  font-weight: 400;
  opacity: 0.8;
}

.reel-title {
  margin: 0 0 4px;
  font-size: 18px;
  font-weight: 600;
}

.reel-location {
  margin: 0 0 14px;
  font-size: 14px;
  color: #ccc;
}

.details-btn {
  width: 100%;
  background: rgba(255, 255, 255, 0.16);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.15);
  color: #fff;
  padding: 13px;
  border-radius: 12px;
  font-size: 15px;
  font-weight: 600;
  display: flex;
  justify-content: center;
  gap: 8px;
  cursor: pointer;
}

.fullscreen-modal {
  position: fixed;
  inset: 0;
  background: #111;
  color: #fff;
  z-index: 200;
  display: flex;
  justify-content: center;
}

.modal-inner {
  width: 100%;
  max-width: 560px;
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: inherit;
}

.detail-header {
  display: flex;
  justify-content: space-between;
  padding: 16px;
}

.back-link, .close-link {
  background: none;
  border: none;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  color: inherit;
}

.close-link {
  color: #007AFF;
}

.detail-scroll {
  flex: 1;
  overflow-y: auto;
  padding-bottom: 90px;
}

.detail-gallery {
  padding: 0 16px;
}

.detail-img {
  width: 100%;
  height: 280px;
  object-fit: cover;
  border-radius: 14px;
}

.detail-body {
  padding: 20px;
}

.detail-price {
  font-size: 30px;
  font-weight: 700;
  margin-top: 10px;
}

.detail-title {
  font-size: 20px;
  margin: 6px 0;
}

.detail-loc {
  color: #8e8e93;
  font-size: 14px;
}

.amenities-row {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-top: 14px;
}

.amenity-chip {
  background: rgba(142, 142, 147, 0.2);
  padding: 6px 12px;
  border-radius: 8px;
  font-size: 13px;
}

.divider {
  border: none;
  border-top: 1px solid #2c2c2e;
  margin: 18px 0;
}

.landlord-box {
  display: flex;
  align-items: center;
  gap: 14px;
  cursor: pointer;
}

.avatar-circle {
  width: 48px;
  height: 48px;
  border-radius: 50%;
  background: #2c2c2e;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 20px;
  font-weight: 700;
}

.avatar-circle.large {
  width: 68px;
  height: 68px;
  font-size: 28px;
  margin: 0 auto 10px;
}

.verified-green {
  color: #34C759;
  font-size: 13px;
  font-weight: 600;
}

.member-sub {
  color: #8e8e93;
  font-size: 13px;
}

.detail-bottom-bar {
  display: flex;
  gap: 12px;
  padding: 14px 16px;
  border-top: 1px solid #2c2c2e;
  background: inherit;
}

.wa-btn, .call-btn {
  flex: 1;
  text-align: center;
  text-decoration: none;
  padding: 13px;
  border-radius: 12px;
  font-weight: 600;
  font-size: 15px;
}

.wa-btn {
  background: #25D366;
  color: #fff;
}

.call-btn {
  background: #2c2c2e;
  color: #fff;
}

.sheet-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.75);
  z-index: 300;
  display: flex;
  justify-content: center;
  align-items: flex-end;
}

.bottom-sheet {
  width: 100%;
  max-width: 560px;
  background: #1C1C1E;
  color: #fff;
  border-radius: 24px 24px 0 0;
  padding: 12px 20px 28px;
  box-sizing: border-box;
  max-height: 90vh;
  overflow-y: auto;
}

.drag-bar {
  width: 44px;
  height: 5px;
  background: rgba(255, 255, 255, 0.3);
  border-radius: 3px;
  margin: 0 auto 14px;
}

.field-label {
  display: block;
  font-size: 13px;
  color: #8e8e93;
  margin: 10px 0 6px;
}

.segment-row {
  display: flex;
  gap: 8px;
  margin-bottom: 12px;
}

.seg-btn {
  flex: 1;
  padding: 11px;
  border-radius: 10px;
  border: 1px solid #3a3a3c;
  background: #2c2c2e;
  color: inherit;
  font-weight: 500;
  cursor: pointer;
}

.seg-btn.active {
  background: #007AFF;
  border-color: #007AFF;
  color: #fff;
}

.seg-btn.orange.active {
  background: #FF9500;
  border-color: #FF9500;
  color: #fff;
}

.sheet-input {
  width: 100%;
  box-sizing: border-box;
  background: #2c2c2e;
  border: 1px solid #3a3a3c;
  border-radius: 10px;
  padding: 12px;
  color: inherit;
  font-size: 15px;
  margin-bottom: 10px;
}

.two-cols {
  display: flex;
  gap: 10px;
}

.primary-action {
  flex: 1;
  background: #007AFF;
  color: #fff;
  border: none;
  padding: 14px;
  border-radius: 12px;
  font-weight: 600;
  font-size: 15px;
  cursor: pointer;
}

.secondary-action {
  flex: 1;
  background: #2c2c2e;
  color: inherit;
  border: 1px solid #3a3a3c;
  padding: 14px;
  border-radius: 12px;
  font-weight: 600;
  cursor: pointer;
}

.full-w {
  width: 100%;
  margin-top: 6px;
}

.profile-top {
  text-align: center;
  margin-bottom: 18px;
}

.mini-apt-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px;
  background: rgba(142, 142, 147, 0.15);
  border-radius: 10px;
  margin-bottom: 8px;
  cursor: pointer;
}

.mini-price {
  color: #FF9500;
  font-weight: 700;
}

.empty-screen {
  height: 100vh;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  color: #fff;
  gap: 14px;
}

.error-banner {
  color: #FF3B30;
  font-size: 13px;
  font-weight: 600;
  text-align: center;
}

.light-theme {
  background: #FFFFFF !important;
  color: #111111 !important;
}
.light-theme .sheet-input,
.light-theme .seg-btn:not(.active),
.light-theme .secondary-action,
.light-theme .call-btn {
  background: #F2F2F7 !important;
  border-color: #E5E5EA !important;
  color: #111111 !important;
}
</style>