<script setup>
import { ref, computed, onMounted, nextTick } from 'vue'

// --- NUTZER / AUTH STATE ---
const currentUser = ref(null) // { id, name, phone } oder null (= Gast)
const authModal = ref(false)
const authName = ref('')
const authPhone = ref('')
const authError = ref('')
const authLoading = ref(false)
const pendingActionAfterLogin = ref(null)

// --- APARTMENTS & UI STATE ---
const apartments = ref([])
const loading = ref(true)
const lang = ref('de')
const isDarkMode = ref(false)
const favorites = ref(new Set())

// Filter & Standort
const activeFeedMode = ref('all')
const userCity = ref(null)
const locationLoading = ref(false)
const filterMinPrice = ref('')
const filterMaxPrice = ref('')
const filterCity = ref('')
const filterType = ref('ALL')
const viewMode = ref('explore') // 'explore', 'reels' oder 'favorites'
const searchQuery = ref('')
const selectedReelId = ref(null)
const reelFeedRef = ref(null)

// Modals
const detailItem = ref(null)
const selectedLandlord = ref(null)
const filterModal = ref(false)
const addModal = ref(false)
const settingsModal = ref(false)
const isPublishing = ref(false)
const formError = ref('')

// Formular für neues Inserat
const newTitle = ref('')
const newPriceRaw = ref('')
const pricePeriod = ref('Monat')
const newType = ref('Langzeit')
const newCity = ref('')
const newDistrict = ref('')
const newSpace = ref('')
const newDesc = ref('')
const newImageData = ref('')
const imageFileName = ref('')

// Automatische Umschaltung zwischen Lokal (npm run dev) und Render (Live)
const BASE_URL = import.meta.env.DEV
  ? 'http://localhost:8080'
  : 'https://student-rent-project.onrender.com'

const API_URL = `${BASE_URL}/api/apartments`
const AUTH_URL = `${BASE_URL}/api/auth/login`

// --- ÜBERSETZUNGEN (DE / RU) ---
const TRANSLATIONS = {
  de: {
    filters: 'Filter',
    nearby: 'In deiner Nähe',
    login: 'Anmelden',
    guest: 'Gast',
    logout: 'Abmelden',
    myAccount: 'Mein Konto',
    loginTitle: 'Konto anmelden',
    loginSubtitle: 'Melde dich mit Name und Handynummer an. Deine Telefonnummer schützt deine eigenen Anzeigen.',
    phoneRequired: 'Bitte Name und gültige Handynummer eingeben!',
    saveAccount: 'Weiter & Anmelden',
    checking: 'Verbinde...',
    favorites: 'Favoriten',
    longTerm: 'Langzeit',
    shortTerm: 'Kurzzeit',
    call: 'Anrufen',
    whatsapp: 'WhatsApp',
    showDetails: 'Details',
    close: 'Schließen',
    delete: 'Löschen',
    apply: 'Filter anwenden',
    reset: 'Zurücksetzen',
    publish: 'Inserat veröffentlichen',
    publishing: 'Wird gespeichert...',
    newListing: 'Inserieren',
    mandatoryError: 'Bitte Titel, Preis und Stadt ausfüllen!',
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
    loading: 'Lade Wohnungen...',
    uploadPhoto: 'Foto aus Galerie / PC auswählen',
    changePhoto: 'Ändern',
    explore: 'Entdecken',
    reels: 'Reels',
    searchPlaceholder: 'Stadt, Bezirk oder Titel suchen',
    all: 'Alle',
    listings: 'Wohnungen',
    noFavoritesTitle: 'Noch nichts gespeichert',
    noFavoritesText: 'Speichere interessante Wohnungen mit dem Herz, um sie später wiederzufinden.',
    emptyTitle: 'Keine passenden Wohnungen gefunden',
    emptyText: 'Passe deine Suche oder Filter an.',
    viewReels: 'Im Reel ansehen',
    backToExplore: 'Übersicht'
  },
  ru: {
    filters: 'Фильтры',
    nearby: 'Рядом',
    login: 'Войти',
    guest: 'Гость',
    logout: 'Выйти',
    myAccount: 'Мой аккаунт',
    loginTitle: 'Вход в аккаунт',
    loginSubtitle: 'Укажите имя и телефон — номер сохраняется в базе и защищает ваши объявления.',
    phoneRequired: 'Введите имя и номер телефона!',
    saveAccount: 'Войти и продолжить',
    checking: 'Вход...',
    favorites: 'Избранное',
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
    mandatoryError: 'Заполните название, цену и город!',
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
    loading: 'Загрузка...',
    uploadPhoto: 'Выбрать фото из галереи / ПК',
    changePhoto: 'Изменить',
    explore: 'Обзор',
    reels: 'Reels',
    searchPlaceholder: 'Город, район или название',
    all: 'Все',
    listings: 'квартир',
    noFavoritesTitle: 'Пока ничего не сохранено',
    noFavoritesText: 'Отмечай понравившиеся квартиры сердечком, чтобы вернуться к ним позже.',
    emptyTitle: 'Подходящих квартир не найдено',
    emptyText: 'Измени поисковый запрос или фильтры.',
    viewReels: 'Открыть Reels',
    backToExplore: 'Обзор'
  }
}

const t = computed(() => TRANSLATIONS[lang.value])

const capitalize = (s) => (s ? s.charAt(0).toUpperCase() + s.slice(1) : s)

// --- FOTO AUS GALERIE / PC KOMPRIMIEREN ---
const handleImageFile = (event) => {
  const file = event.target.files?.[0]
  if (!file) return
  imageFileName.value = file.name

  const reader = new FileReader()
  reader.onload = (e) => {
    const img = new Image()
    img.onload = () => {
      const canvas = document.createElement('canvas')
      const maxWidth = 900
      const scale = Math.min(1, maxWidth / img.width)
      canvas.width = img.width * scale
      canvas.height = img.height * scale

      const ctx = canvas.getContext('2d')
      ctx.drawImage(img, 0, 0, canvas.width, canvas.height)
      newImageData.value = canvas.toDataURL('image/jpeg', 0.8)
    }
    img.src = e.target.result
  }
  reader.readAsDataURL(file)
}

// --- LOGIN NUR ÜBER TELEFONNUMMER GEGEN POSTGRESQL ---
const handleLoginSubmit = async () => {
  if (!authName.value.trim() || authPhone.value.trim().length < 6) {
    authError.value = t.value.phoneRequired
    return
  }

  authLoading.value = true
  authError.value = ''

  try {
    const res = await fetch(AUTH_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        name: authName.value.trim(),
        phone: authPhone.value.trim()
      })
    })

    if (!res.ok) {
      authError.value = t.value.phoneRequired
      return
    }

    const userObj = await res.json()
    currentUser.value = userObj
    localStorage.setItem('studentrent_user', JSON.stringify(userObj))
    authModal.value = false

    if (pendingActionAfterLogin.value === 'add') {
      pendingActionAfterLogin.value = null
      addModal.value = true
    }
  } catch {
    authError.value = 'Verbindungsfehler zum Server.'
  } finally {
    authLoading.value = false
  }
}

const handleLogout = () => {
  currentUser.value = null
  localStorage.removeItem('studentrent_user')
  settingsModal.value = false
}

const openAddListingFlow = () => {
  if (!currentUser.value) {
    pendingActionAfterLogin.value = 'add'
    authError.value = ''
    authModal.value = true
  } else {
    addModal.value = true
  }
}

const isOwnListing = (item) => {
  if (!currentUser.value || !item?.phone) return false
  const itemPhone = item.phone.replace(/[^0-9+]/g, '')
  const userPhone = (currentUser.value.phone || '').replace(/[^0-9+]/g, '')
  return itemPhone !== '' && itemPhone === userPhone
}

const sanitizePrice = (val) => String(val ?? '').replace(/[^0-9]/g, '')

// --- ZEITBERECHNUNG (AUTOMATISCHER ZEITZONEN-AUSGLEICH) ---
const getRelativeTimeString = (isoTimestamp) => {
  if (!isoTimestamp) return lang.value === 'de' ? 'Neu' : 'Новое'
  const str = String(isoTimestamp).trim()
  const now = Date.now()

  const localMs = new Date(str).getTime()
  const utcMs = new Date(str.endsWith('Z') || /[+-]\d{2}:?\d{2}$/.test(str) ? str : `${str}Z`).getTime()

  const diffLocal = now - localMs
  const diffUtc = now - utcMs
  const bestDiffMs =
    diffUtc >= -60000 && Math.abs(diffUtc) < Math.abs(diffLocal) ? diffUtc : diffLocal

  if (Number.isNaN(bestDiffMs)) return lang.value === 'de' ? 'Neu' : 'Новое'

  const diffInMinutes = Math.max(0, Math.floor(bestDiffMs / (1000 * 60)))
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

// --- 1. GET: DATEN VOM BACKEND LADEN ---
const fetchListings = async () => {
  loading.value = true
  try {
    const res = await fetch(API_URL)
    const data = await res.json()
    apartments.value = [...data].sort((a, b) => (b.id || 0) - (a.id || 0))
  } catch (err) {
    console.error('Fehler beim Laden:', err)
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

// --- COMPUTED LISTS ---
const filteredApartments = computed(() => {
  const query = searchQuery.value.trim().toLowerCase()
  return apartments.value.filter((apt) => {
    if (activeFeedMode.value === 'nearby' && userCity.value) {
      if (apt.city?.toLowerCase() !== userCity.value.toLowerCase()) return false
    }
    if (filterType.value !== 'ALL' && (apt.type || 'Langzeit') !== filterType.value) return false
    if (filterCity.value && !apt.city?.toLowerCase().includes(filterCity.value.toLowerCase())) return false
    if (filterMinPrice.value && apt.price < parseFloat(filterMinPrice.value)) return false
    if (filterMaxPrice.value && apt.price > parseFloat(filterMaxPrice.value)) return false
    if (query) {
      const searchable = [apt.title, apt.city, apt.district, apt.description].filter(Boolean).join(' ').toLowerCase()
      if (!searchable.includes(query)) return false
    }
    return true
  })
})

const displayedApartments = computed(() => {
  if (viewMode.value === 'favorites') {
    return filteredApartments.value.filter((apt) => favorites.value.has(apt.id))
  }
  return filteredApartments.value
})

const reelApartments = computed(() => {
  const list = filteredApartments.value
  const startIndex = list.findIndex((apt) => apt.id === selectedReelId.value)
  if (startIndex <= 0) return list
  return [...list.slice(startIndex), ...list.slice(0, startIndex)]
})

const enterReels = async (apt = null) => {
  selectedReelId.value = apt?.id ?? null
  viewMode.value = 'reels'
  await nextTick()
  if (reelFeedRef.value) reelFeedRef.value.scrollTop = 0
}

const setViewMode = (mode) => {
  viewMode.value = mode
  if (mode === 'reels') selectedReelId.value = null
}

const favoriteApartments = computed(() => {
  return apartments.value.filter((apt) => favorites.value.has(apt.id))
})

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

// --- 2. POST: NEUES INSERAT SPEICHERN ---
const handleAddListing = async () => {
  if (!currentUser.value) {
    addModal.value = false
    authModal.value = true
    return
  }

  if (!newTitle.value.trim() || !newPriceRaw.value || !newCity.value.trim()) {
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
    city: capitalize(newCity.value.trim()),
    district: capitalize(newDistrict.value.trim()) || 'Zentrum',
    space: newSpace.value ? `${newSpace.value} m²` : null,
    phone: currentUser.value.phone,
    description: newDesc.value.trim() || null,
    imageUrl: newImageData.value || null,
    landlordName: currentUser.value.name,
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
      newDesc.value = ''
      newType.value = 'Langzeit'
      pricePeriod.value = 'Monat'
      newImageData.value = ''
      imageFileName.value = ''
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

// --- 3. DELETE: NUR EIGENE INSERATE LÖSCHEN ---
const handleDelete = async (item) => {
  if (!isOwnListing(item)) return
  if (!confirm(lang.value === 'de' ? 'Möchtest du dein Inserat wirklich löschen?' : 'Удалить это объявление?')) return

  try {
    const res = await fetch(`${API_URL}/${item.id}`, { method: 'DELETE' })
    if (res.ok) {
      apartments.value = apartments.value.filter((a) => a.id !== item.id)
      favorites.value.delete(item.id)
      saveFavorites()
      if (detailItem.value?.id === item.id) detailItem.value = null
    }
  } catch (err) {
    console.error('Fehler beim Löschen:', err)
  }
}

// --- FAVORITEN SPEICHERN ---
const saveFavorites = () => {
  localStorage.setItem('studentrent_favs', JSON.stringify(Array.from(favorites.value)))
}

const toggleFavorite = (id) => {
  if (favorites.value.has(id)) favorites.value.delete(id)
  else favorites.value.add(id)
  saveFavorites()
}

const resetFilters = () => {
  filterMinPrice.value = ''
  filterMaxPrice.value = ''
  filterCity.value = ''
  filterType.value = 'ALL'
  searchQuery.value = ''
  activeFeedMode.value = 'all'
  filterModal.value = false
}

onMounted(() => {
  const savedUser = localStorage.getItem('studentrent_user')
  if (savedUser) {
    try {
      currentUser.value = JSON.parse(savedUser)
    } catch {}
  }
  const savedFavs = localStorage.getItem('studentrent_favs')
  if (savedFavs) {
    try {
      favorites.value = new Set(JSON.parse(savedFavs))
    } catch {}
  }
  fetchListings()
})
</script>

<template>
  <div class="app-root" :class="{ 'theme-dark': isDarkMode }">
    <header class="site-header">
      <div class="header-inner">
        <button class="brand" type="button" @click="setViewMode('explore')" aria-label="Zur Startseite">
          <span class="brand-mark">SR</span>
          <span class="brand-name">student rent</span>
        </button>

        <nav class="desktop-nav" aria-label="Hauptnavigation">
          <button type="button" :class="{ active: viewMode === 'explore' }" @click="setViewMode('explore')">{{ t.explore }}</button>
          <button type="button" :class="{ active: viewMode === 'reels' }" @click="enterReels()">{{ t.reels }}</button>
          <button type="button" :class="{ active: viewMode === 'favorites' }" @click="setViewMode('favorites')">{{ t.favorites }}<span v-if="favoriteApartments.length" class="nav-count">{{ favoriteApartments.length }}</span></button>
        </nav>

        <div class="header-actions">
          <button class="nearby-button" :class="{ selected: activeFeedMode === 'nearby' }" type="button" @click="toggleNearby">
            <span class="button-icon">⌖</span>
            <span>{{ locationLoading ? '…' : t.nearby }}</span>
          </button>
          <button class="icon-button settings-trigger" type="button" @click="settingsModal = true" :title="t.settings" aria-label="Einstellungen">⚙</button>
          <button class="account-button" type="button" @click="currentUser ? (settingsModal = true) : (authModal = true)">
            <span class="account-avatar">{{ currentUser ? currentUser.name.charAt(0).toUpperCase() : '♙' }}</span>
            <span class="account-label">{{ currentUser ? currentUser.name : t.login }}</span>
          </button>
          <button class="publish-button" type="button" @click="openAddListingFlow"><span>＋</span>{{ t.newListing }}</button>
        </div>
      </div>
    </header>

    <Transition name="view" mode="out-in">
    <main v-if="viewMode !== 'reels'" :key="viewMode" class="page-main">
      <section class="search-panel" aria-label="Wohnungssuche">
        <label class="search-field">
          <span class="search-icon">⌕</span>
          <input v-model="searchQuery" type="search" :placeholder="t.searchPlaceholder" />
          <button v-if="searchQuery" class="clear-search" type="button" @click="searchQuery = ''" aria-label="Suche löschen">×</button>
        </label>
        <button class="filter-open-button" type="button" @click="filterModal = true"><span>☷</span>{{ t.filters }}</button>
        <button class="search-nearby" type="button" :class="{ selected: activeFeedMode === 'nearby' }" @click="toggleNearby"><span>⌖</span>{{ t.nearby }}</button>
      </section>

      <section class="listing-toolbar">
        <div :class="['type-tabs', { 'is-langzeit': filterType === 'Langzeit', 'is-kurzzeit': filterType === 'Kurzzeit' }]" role="group" :aria-label="t.rentalPeriod">
          <button type="button" :class="{ active: filterType === 'ALL' }" @click="filterType = 'ALL'">{{ t.all }}</button>
          <button type="button" :class="{ active: filterType === 'Langzeit' }" @click="filterType = 'Langzeit'">{{ t.longTerm }}</button>
          <button type="button" :class="{ active: filterType === 'Kurzzeit' }" @click="filterType = 'Kurzzeit'">{{ t.shortTerm }}</button>
        </div>
        <div class="results-meta">
          <span class="results-number">{{ displayedApartments.length }}</span>
          <span>{{ t.listings }}</span>
        </div>
      </section>

      <section v-if="loading" class="state-panel">
        <div class="loading-orbit"></div>
        <h2>{{ t.loading }}</h2>
      </section>

      <TransitionGroup v-else-if="displayedApartments.length" name="listing" tag="section" class="listing-grid">
        <article v-for="item in displayedApartments" :key="item.id" class="listing-card">
          <div class="listing-image-wrap" @click="enterReels(item)" role="button" tabindex="0" @keydown.enter="enterReels(item)" :aria-label="t.viewReels">
            <img :src="item.imageUrl || DEFAULT_FALLBACK_IMAGE" class="listing-image" alt="Wohnungsfoto" loading="lazy" />
            <div class="image-shade"></div>
            <span class="listing-type-tag">{{ (item.type || 'Langzeit') === 'Langzeit' ? t.longTerm : t.shortTerm }}</span>

            <button
              v-if="isOwnListing(item)"
              class="card-delete-btn"
              type="button"
              @click.stop="handleDelete(item)"
              :title="t.delete"
            >
              🗑
            </button>

            <button class="favorite-toggle" type="button" @click.stop="toggleFavorite(item.id)" :aria-label="favorites.has(item.id) ? 'Aus Favoriten entfernen' : 'Zu Favoriten hinzufügen'" :class="{ liked: favorites.has(item.id) }">
              {{ favorites.has(item.id) ? '♥' : '♡' }}
            </button>

            <div class="price-sticker">
              <strong>{{ item.price }} €</strong>
              <small>{{ item.priceSuffix || t.perMonth }}</small>
            </div>
          </div>

          <div class="listing-content">
            <button class="listing-title" type="button" @click="detailItem = item">{{ item.title }}</button>
            <div class="listing-location">
              <span>⌖</span>{{ capitalize(item.city) }}<template v-if="item.district">, {{ capitalize(item.district) }}</template>
              <em v-if="item.space" class="area-pill">{{ item.space }}</em>
            </div>
            <div class="listing-card-footer">
              <button class="landlord-mini" type="button" @click="openLandlordProfile(item)">
                <span class="mini-avatar">{{ (item.landlordName || 'S').charAt(0).toUpperCase() }}</span>
                <span class="landlord-name">{{ item.landlordName || 'Student_Vermieter' }}</span>
                <span v-if="isOwnListing(item)" class="own-badge">(Du)</span>
                <span class="listing-age">{{ getRelativeTimeString(item.createdAt) }}</span>
              </button>
              <button class="details-link" type="button" @click="detailItem = item">{{ t.showDetails }} ↗</button>
            </div>
          </div>
        </article>
      </TransitionGroup>

      <section v-else class="state-panel empty-state">
        <div class="empty-icon">⌂</div>
        <h2>{{ viewMode === 'favorites' ? t.noFavoritesTitle : t.emptyTitle }}</h2>
        <p>{{ viewMode === 'favorites' ? t.noFavoritesText : t.emptyText }}</p>
        <button v-if="viewMode === 'favorites'" class="primary-button" type="button" @click="setViewMode('explore')">{{ t.explore }}</button>
        <button v-else class="secondary-button" type="button" @click="resetFilters">{{ t.reset }}</button>
      </section>

      <footer class="site-footer"></footer>
    </main>

    <!-- REELS MODUS -->
    <main v-else :key="viewMode" class="reels-page">
      <div class="reels-heading">
        <button type="button" class="back-to-catalog" @click="setViewMode('explore')">← {{ t.backToExplore }}</button>
        <div class="reels-heading-center"><span class="reels-live-dot"></span>{{ t.reels }} <span class="reels-counter">{{ reelApartments.length }} {{ t.listings }}</span></div>
        <button type="button" class="reels-filter-button" @click="filterModal = true">☷ {{ t.filters }}</button>
      </div>

      <div v-if="!loading && !reelApartments.length" class="state-panel empty-state reels-empty">
        <div class="empty-icon">⌂</div><h2>{{ t.emptyTitle }}</h2><p>{{ t.emptyText }}</p>
        <button class="secondary-button" type="button" @click="setViewMode('explore')">{{ t.backToExplore }}</button>
      </div>
      <div v-else-if="loading" class="state-panel"><div class="loading-orbit"></div><h2>{{ t.loading }}</h2></div>

      <div v-else ref="reelFeedRef" class="reels-feed">
        <section v-for="item in reelApartments" :key="item.id" class="reel-card">
          <img :src="item.imageUrl || DEFAULT_FALLBACK_IMAGE" class="reel-image" alt="Wohnungsfoto" />
          <div class="reel-gradient"></div>

          <div class="reel-top-tags">
            <span class="listing-type-tag">{{ (item.type || 'Langzeit') === 'Langzeit' ? t.longTerm : t.shortTerm }}</span>
            <span class="reel-date-tag">{{ getRelativeTimeString(item.createdAt) }}</span>
          </div>

          <div class="right-floating-bar">
            <button v-if="isOwnListing(item)" class="float-circle delete-own" type="button" @click="handleDelete(item)" :title="t.delete">🗑</button>
            <button class="float-circle" type="button" @click="toggleFavorite(item.id)" :class="{ liked: favorites.has(item.id) }" :aria-label="t.favorites">{{ favorites.has(item.id) ? '♥' : '♡' }}</button>
            <button class="float-circle" type="button" @click="openLandlordProfile(item)" aria-label="Vermieterprofil">♙</button>
          </div>

          <div class="reel-bottom">
            <div class="reel-info">
              <div class="reel-location">
                ⌖ {{ capitalize(item.city) }}<template v-if="item.district">, {{ capitalize(item.district) }}</template>
              </div>
              <h2 class="reel-title">{{ item.title }}</h2>
              <div class="reel-meta-row">
                <button class="landlord-chip" type="button" @click="openLandlordProfile(item)">
                  <span class="mini-avatar">{{ (item.landlordName || 'S').charAt(0).toUpperCase() }}</span>
                  <span>{{ item.landlordName || 'Student_Vermieter' }}</span>
                  <span v-if="isOwnListing(item)" class="own-tag">(Du)</span>
                </button>
                <span v-if="item.space" class="reel-space-pill">{{ item.space }}</span>
              </div>
            </div>

            <div class="reel-side">
              <div class="price-sticker big">
                <strong>{{ item.price }} €</strong>
                <small>{{ item.priceSuffix || t.perMonth }}</small>
              </div>
              <button class="reel-details-button" type="button" @click="detailItem = item">{{ t.showDetails }} ↗</button>
            </div>
          </div>
        </section>
      </div>
      <div class="reels-footnote">{{ lang === 'de' ? 'Wische oder scrolle für weitere Wohnungen' : 'Листай вверх и вниз для других квартир' }}</div>
    </main>
    </Transition>

    <nav class="mobile-bottom-nav" aria-label="Mobile Navigation">
      <button type="button" :class="{ active: viewMode === 'explore' }" @click="setViewMode('explore')"><span>⌂</span>{{ t.explore }}</button>
      <button type="button" :class="{ active: viewMode === 'reels' }" @click="enterReels()"><span>▤</span>{{ t.reels }}</button>
      <button type="button" :class="{ active: viewMode === 'favorites' }" @click="setViewMode('favorites')"><span>♡</span>{{ t.favorites }}<i v-if="favoriteApartments.length">{{ favoriteApartments.length }}</i></button>
      <button type="button" @click="settingsModal = true"><span>♙</span>{{ t.myAccount }}</button>
    </nav>

    <!-- Login -->
    <div v-if="authModal" class="sheet-backdrop" @click.self="authModal = false">
      <form class="bottom-sheet" @submit.prevent="handleLoginSubmit">
        <div class="drag-bar"></div>
        <h3>{{ t.loginTitle }}</h3>
        <p class="member-sub">{{ t.loginSubtitle }}</p>
        <p v-if="authError" class="error-banner">{{ authError }}</p>
        <label class="form-label">{{ lang === 'de' ? 'Handynummer' : 'Номер телефона' }} *</label>
        <input v-model="authPhone" type="tel" placeholder="+49 170 1234567" class="sheet-input" autocomplete="tel" required />
        <label class="form-label">{{ lang === 'de' ? 'Dein Name' : 'Твоё имя' }} *</label>
        <input v-model="authName" :placeholder="lang === 'de' ? 'Dein Name' : 'Твоё имя'" class="sheet-input" autocomplete="name" required />
        <button class="primary-button full-w" type="submit" :disabled="authLoading">
          {{ authLoading ? t.checking : t.saveAccount }}
        </button>
      </form>
    </div>

    <!-- Apartment details -->
    <div v-if="detailItem" class="detail-overlay">
      <div class="detail-page">
        <header class="detail-header">
          <button class="back-link" type="button" @click="detailItem = null">← {{ capitalize(detailItem.city) }}</button>
          <div class="detail-header-actions">
            <button
              v-if="isOwnListing(detailItem)"
              class="delete-text-btn"
              type="button"
              @click="handleDelete(detailItem)"
            >
              🗑 {{ t.delete }}
            </button>
            <button class="close-link" type="button" @click="detailItem = null">{{ t.close }} ✕</button>
          </div>
        </header>
        <div class="detail-scroll">
          <div class="detail-layout">
            <div class="detail-cover-wrap">
              <img :src="detailItem.imageUrl || DEFAULT_FALLBACK_IMAGE" class="detail-img" alt="Wohnungsfoto" />
              <span class="detail-type-tag">{{ (detailItem.type || 'Langzeit') === 'Langzeit' ? t.longTerm : t.shortTerm }}</span>
              <button class="detail-favorite" type="button" @click="toggleFavorite(detailItem.id)">{{ favorites.has(detailItem.id) ? '♥' : '♡' }}</button>
            </div>
            <div class="detail-body">
              <div class="price-sticker big detail-sticker">
                <strong>{{ detailItem.price }} €</strong>
                <small>{{ detailItem.priceSuffix || t.perMonth }}</small>
              </div>
              <h1 class="detail-title">{{ detailItem.title }}</h1>
              <p class="detail-loc">⌖ {{ capitalize(detailItem.city) }}<template v-if="detailItem.district">, {{ capitalize(detailItem.district) }}</template> · {{ getRelativeTimeString(detailItem.createdAt) }}</p>
              <div v-if="detailItem.space" class="detail-facts"><span>⌂ {{ detailItem.space }}</span><span>{{ (detailItem.type || 'Langzeit') === 'Langzeit' ? t.longTerm : t.shortTerm }}</span></div>
              <button class="landlord-profile-card" type="button" @click="openLandlordProfile(detailItem)">
                <span class="avatar-circle">{{ (detailItem.landlordName || 'S').charAt(0).toUpperCase() }}</span>
                <span class="landlord-info"><strong>{{ detailItem.landlordName || 'Student_Vermieter' }}</strong><small>{{ t.memberSince }} {{ detailItem.memberSince || '2026' }}</small></span>
                <span class="profile-arrow">↗</span>
              </button>
              <h3>{{ t.description }}</h3>
              <p class="desc-text">{{ detailItem.description || (lang === 'de' ? 'Keine zusätzliche Beschreibung angegeben.' : 'Дополнительное описание не указано.') }}</p>
            </div>
          </div>
        </div>
        <div v-if="detailItem.phone" class="detail-bottom-bar">
          <a :href="`https://wa.me/${detailItem.phone.replace(/[^0-9]/g, '')}`" target="_blank" rel="noopener" class="wa-btn">◉ WhatsApp</a>
          <a :href="`tel:${detailItem.phone}`" class="call-btn">☎ {{ t.call }}</a>
        </div>
      </div>
    </div>

    <!-- Landlord profile -->
    <div v-if="selectedLandlord" class="sheet-backdrop" @click.self="selectedLandlord = null">
      <div class="bottom-sheet profile-sheet">
        <div class="drag-bar"></div>
        <div class="profile-top">
          <div class="avatar-circle large">{{ selectedLandlord.name.charAt(0).toUpperCase() }}</div>
          <h2>{{ selectedLandlord.name }}</h2>
          <p class="profile-member">{{ t.memberSince }} {{ selectedLandlord.memberSince }}</p>
        </div>
        <h4>{{ t.otherListings }} {{ selectedLandlord.name }} ({{ landlordListings.length }})</h4>
        <div v-if="landlordListings.length" class="mini-list">
          <button v-for="apt in landlordListings" :key="apt.id" class="mini-apt-row" type="button" @click="selectedLandlord = null; detailItem = apt">
            <img :src="apt.imageUrl || DEFAULT_FALLBACK_IMAGE" class="fav-thumb" alt="" />
            <span class="mini-apt-info"><strong>{{ apt.title }}</strong><small>{{ capitalize(apt.city) }}, {{ capitalize(apt.district) }}</small></span>
            <span class="mini-price">{{ apt.price }} €</span>
          </button>
        </div>
        <p v-else class="member-sub">{{ t.noResults }}</p>
      </div>
    </div>

    <!-- Filters -->
    <div v-if="filterModal" class="sheet-backdrop" @click.self="filterModal = false">
      <div class="bottom-sheet">
        <div class="drag-bar"></div>
        <h3>{{ t.filters }}</h3>
        <label class="form-label">{{ t.rentalPeriod }}</label>
        <div class="segment-row">
          <button :class="['seg-btn', { active: filterType === 'ALL' }]" type="button" @click="filterType = 'ALL'">{{ t.all }}</button>
          <button :class="['seg-btn', { active: filterType === 'Langzeit' }]" type="button" @click="filterType = 'Langzeit'">{{ t.longTerm }}</button>
          <button :class="['seg-btn', { active: filterType === 'Kurzzeit' }]" type="button" @click="filterType = 'Kurzzeit'">{{ t.shortTerm }}</button>
        </div>
        <label class="form-label">{{ lang === 'de' ? 'Monatlicher / täglicher Preis (€)' : 'Цена (€)' }}</label>
        <div class="two-cols">
          <input :value="filterMinPrice" @input="filterMinPrice = sanitizePrice($event.target.value)" inputmode="numeric" :placeholder="lang === 'de' ? 'Ab €' : 'От €'" class="sheet-input" />
          <input :value="filterMaxPrice" @input="filterMaxPrice = sanitizePrice($event.target.value)" inputmode="numeric" :placeholder="lang === 'de' ? 'Bis €' : 'До €'" class="sheet-input" />
        </div>
        <label class="form-label">{{ lang === 'de' ? 'Stadt' : 'Город' }}</label>
        <input v-model="filterCity" :placeholder="lang === 'de' ? 'z. B. Berlin' : 'Например, Berlin'" class="sheet-input" />
        <div class="sheet-actions">
          <button class="secondary-button" type="button" @click="resetFilters">{{ t.reset }}</button>
          <button class="primary-button" type="button" @click="filterModal = false">{{ t.apply }}</button>
        </div>
      </div>
    </div>

    <!-- Settings and account -->
    <div v-if="settingsModal" class="sheet-backdrop" @click.self="settingsModal = false">
      <div class="bottom-sheet">
        <div class="drag-bar"></div>
        <h3>{{ t.settings }}</h3>
        <div class="account-card">
          <template v-if="currentUser">
            <div class="account-card-user"><span class="account-avatar large-avatar">{{ currentUser.name.charAt(0).toUpperCase() }}</span><span><strong>{{ currentUser.name }}</strong><small>{{ currentUser.phone }}</small></span></div>
            <button class="logout-btn" type="button" @click="handleLogout">{{ t.logout }}</button>
          </template>
          <div v-else class="guest-account"><span>{{ t.guest }}</span><button class="primary-button compact" type="button" @click="settingsModal = false; authModal = true">{{ t.login }}</button></div>
        </div>
        <label class="form-label">{{ t.theme }}</label>
        <div class="segment-row">
          <button :class="['seg-btn', { active: !isDarkMode }]" type="button" @click="isDarkMode = false">☀ {{ t.lightMode }}</button>
          <button :class="['seg-btn', { active: isDarkMode }]" type="button" @click="isDarkMode = true">☾ {{ t.darkMode }}</button>
        </div>
        <label class="form-label">{{ t.language }}</label>
        <div class="segment-row">
          <button :class="['seg-btn', { active: lang === 'de' }]" type="button" @click="lang = 'de'">🇩🇪 Deutsch</button>
          <button :class="['seg-btn', { active: lang === 'ru' }]" type="button" @click="lang = 'ru'">🇷🇺 Русский</button>
        </div>
      </div>
    </div>

    <!-- Create listing -->
    <div v-if="addModal" class="sheet-backdrop" @click.self="!isPublishing && (addModal = false)">
      <form class="bottom-sheet listing-form" @submit.prevent="handleAddListing">
        <div class="drag-bar"></div>
        <h3>{{ t.newListing }}</h3>
        <p class="member-sub publisher-line">{{ lang === 'de' ? 'Veröffentlicht als' : 'Публикуется от имени' }} <strong>{{ currentUser?.name }}</strong><span> · {{ currentUser?.phone }}</span></p>
        <p v-if="formError" class="error-banner">{{ formError }}</p>
        <label class="form-label">{{ t.rentalPeriod }}</label>
        <div class="segment-row">
          <button :class="['seg-btn', { active: newType === 'Langzeit' }]" type="button" @click="newType = 'Langzeit'">{{ t.longTerm }}</button>
          <button :class="['seg-btn', { active: newType === 'Kurzzeit' }]" type="button" @click="newType = 'Kurzzeit'">{{ t.shortTerm }}</button>
        </div>
        <label class="form-label">{{ lang === 'de' ? 'Titel der Anzeige' : 'Название объявления' }} *</label>
        <input v-model="newTitle" :placeholder="lang === 'de' ? 'z. B. Helles Apartment nahe Zentrum' : 'Например, светлая квартира в центре'" class="sheet-input" required />
        <div class="form-grid two-cols">
          <div class="form-field"><label class="form-label">{{ lang === 'de' ? 'Preis (€)' : 'Цена (€)' }} *</label><input :value="newPriceRaw" @input="newPriceRaw = sanitizePrice($event.target.value)" inputmode="numeric" placeholder="z. B. 850" class="sheet-input" required /></div>
          <div class="form-field"><label class="form-label">{{ lang === 'de' ? 'Abrechnung' : 'Период' }}</label><div class="segment-row period-segment"><button :class="['seg-btn', { active: pricePeriod === 'Monat' }]" type="button" @click="pricePeriod = 'Monat'">{{ t.perMonth }}</button><button :class="['seg-btn', { active: pricePeriod === 'Tag' }]" type="button" @click="pricePeriod = 'Tag'">{{ t.perDay }}</button></div></div>
        </div>
        <div class="form-grid two-cols">
          <div class="form-field"><label class="form-label">{{ lang === 'de' ? 'Stadt' : 'Город' }} *</label><input v-model="newCity" placeholder="Berlin" class="sheet-input" required /></div>
          <div class="form-field"><label class="form-label">{{ lang === 'de' ? 'Bezirk' : 'Район' }}</label><input v-model="newDistrict" :placeholder="lang === 'de' ? 'z. B. Mitte' : 'Например, Mitte'" class="sheet-input" /></div>
        </div>
        <label class="form-label">{{ lang === 'de' ? 'Wohnfläche (m²)' : 'Площадь (м²)' }}</label>
        <input :value="newSpace" @input="newSpace = sanitizePrice($event.target.value)" inputmode="numeric" placeholder="z. B. 52" class="sheet-input" />
        <label class="form-label">{{ lang === 'de' ? 'Fotos' : 'Фотографии' }}</label>
        <label class="upload-box">
          <input type="file" accept="image/*" @change="handleImageFile" />
          <span class="upload-icon">＋</span>
          <span><strong>{{ newImageData ? imageFileName : t.uploadPhoto }}</strong><small>{{ newImageData ? t.changePhoto : (lang === 'de' ? 'JPG oder PNG auswählen' : 'Выберите JPG или PNG') }}</small></span>
        </label>
        <img v-if="newImageData" :src="newImageData" class="upload-preview" alt="Vorschau" />
        <label class="form-label">{{ t.description }}</label>
        <textarea v-model="newDesc" :placeholder="lang === 'de' ? 'Beschreibe die Wohnung, Ausstattung und Besonderheiten …' : 'Расскажи о квартире, оснащении и особенностях …'" rows="4" class="sheet-input description-input"></textarea>
        <button class="primary-button full-w publish-submit" type="submit" :disabled="isPublishing">{{ isPublishing ? t.publishing : t.publish }}</button>
      </form>
    </div>
  </div>
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=Bricolage+Grotesque:opsz,wght@12..96,500;12..96,700;12..96,800&family=Figtree:wght@400;500;600;700&display=swap');
* { box-sizing: border-box; }
html, body, #app { margin: 0; min-height: 100%; width: 100%; font-family: 'Figtree', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; }
body { background: #eff1f7; }
button, input, textarea { font: inherit; }
button { -webkit-tap-highlight-color: transparent; }
</style>

<style scoped>
/* ---------- Design-Tokens ---------- */
.app-root {
  --page: #eff1f7;
  --surface: #ffffff;
  --surface-soft: #e4e8f5;
  --text: #151a33;
  --muted: #666d8f;
  --line: #d9deee;
  --brand: #3347f5;
  --brand-soft: #e1e5ff;
  --on-brand: #ffffff;
  --sticker: #b8e6d0;
  --sticker-ink: #14362b;
  --danger: #e5484d;
  --display: 'Bricolage Grotesque', 'Figtree', sans-serif;
  min-height: 100vh;
  background: var(--page);
  color: var(--text);
  transition: background .2s ease, color .2s ease;
  padding-bottom: 24px;
}
.app-root.theme-dark {
  --page: #0f1226;
  --surface: #181c38;
  --surface-soft: #232851;
  --text: #f2f3fa;
  --muted: #9aa1c7;
  --line: #2b3160;
  --brand: #93a0ff;
  --brand-soft: #262c5e;
  --on-brand: #0f1226;
  --sticker: #8fd9b8;
  --sticker-ink: #0d281f;
  --danger: #ff7a7f;
}

/* ---------- Header ---------- */
.site-header { position: sticky; top: 0; z-index: 80; background: color-mix(in srgb, var(--page) 88%, transparent); backdrop-filter: blur(16px); }
.header-inner { width: min(1280px, calc(100% - 56px)); height: 78px; margin: 0 auto; display: flex; align-items: center; gap: 36px; }
.brand { display: inline-flex; align-items: center; gap: 10px; background: none; border: 0; padding: 0; color: var(--text); cursor: pointer; flex-shrink: 0; }
.brand-mark { display: grid; place-items: center; width: 36px; height: 36px; border-radius: 11px; background: var(--sticker); color: var(--sticker-ink); font: 800 13px var(--display); }
.brand-name { font: 800 22px var(--display); letter-spacing: -.8px; white-space: nowrap; }
.desktop-nav { display: flex; align-items: center; gap: 4px; padding: 4px; border-radius: 999px; background: var(--surface); border: 1px solid var(--line); }
.desktop-nav button { display: inline-flex; align-items: center; border: 0; border-radius: 999px; background: none; color: var(--muted); font-size: 14px; font-weight: 600; cursor: pointer; padding: 9px 18px; transition: background .2s ease, color .2s ease; }
.desktop-nav button:hover { color: var(--text); }
.desktop-nav button.active { background: var(--text); color: var(--page); }
.nav-count { display: inline-grid; place-items: center; min-width: 18px; height: 18px; margin-left: 7px; padding: 0 5px; border-radius: 9px; background: var(--sticker); color: var(--sticker-ink); font-size: 10px; font-weight: 800; }
.header-actions { display: flex; align-items: center; gap: 10px; margin-left: auto; }
.nearby-button, .search-nearby, .filter-open-button { display: inline-flex; align-items: center; justify-content: center; gap: 8px; border: 1px solid var(--line); border-radius: 999px; padding: 10px 16px; background: var(--surface); color: var(--text); font-weight: 600; font-size: 13px; cursor: pointer; white-space: nowrap; }
.nearby-button.selected, .search-nearby.selected { background: var(--brand); border-color: var(--brand); color: var(--on-brand); }
.button-icon { font-size: 18px; line-height: 12px; }
.icon-button { width: 40px; height: 40px; display: grid; place-items: center; border: 1px solid var(--line); border-radius: 50%; background: var(--surface); color: var(--text); cursor: pointer; font-size: 17px; }
.account-button { border: 0; background: none; display: flex; align-items: center; gap: 8px; color: var(--text); font-size: 13px; font-weight: 600; cursor: pointer; padding: 3px 5px; max-width: 150px; }
.account-avatar { display: grid; place-items: center; width: 34px; height: 34px; border-radius: 50%; background: var(--brand-soft); color: var(--brand); font-size: 14px; font-weight: 800; flex-shrink: 0; }
.account-label { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.publish-button, .primary-button { display: inline-flex; align-items: center; justify-content: center; gap: 8px; padding: 12px 20px; border: 0; border-radius: 999px; background: var(--brand); color: var(--on-brand); font-size: 13px; font-weight: 700; cursor: pointer; transition: transform .18s ease, opacity .18s ease; }
.publish-button:hover, .primary-button:hover { transform: translateY(-1px); }
.publish-button span { font-size: 18px; line-height: 12px; }
.nearby-button, .search-nearby, .filter-open-button, .icon-button, .secondary-button, .favorite-toggle, .detail-favorite { transition: background-color .2s ease, border-color .2s ease, color .2s ease, transform .2s ease; }
.nearby-button:hover, .search-nearby:hover, .filter-open-button:hover, .icon-button:hover, .secondary-button:hover { transform: translateY(-1px); }

/* ---------- Übersicht ---------- */
.page-main { width: min(1280px, calc(100% - 56px)); margin: 0 auto; }
.search-panel { margin-top: 24px; display: flex; gap: 8px; padding: 8px; border-radius: 999px; border: 1px solid var(--line); background: var(--surface); }
.search-field { display: flex; align-items: center; gap: 12px; flex: 1; min-width: 0; padding: 0 18px; }
.search-icon { font-size: 25px; color: var(--muted); line-height: 1; transform: rotate(-20deg); }
.search-field input { width: 100%; min-width: 0; border: 0; outline: 0; background: transparent; color: var(--text); font-size: 15px; }
.search-field input::placeholder { color: var(--muted); }
.clear-search { border: 0; background: var(--surface-soft); color: var(--muted); width: 26px; height: 26px; border-radius: 50%; cursor: pointer; }
.filter-open-button, .search-nearby { padding: 13px 20px; background: var(--page); }
.listing-toolbar { display: flex; align-items: center; justify-content: space-between; gap: 18px; padding: 30px 0 22px; }
.type-tabs { position: relative; isolation: isolate; display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); align-items: center; padding: 4px; background: var(--surface-soft); border-radius: 999px; }
.type-tabs::before { content: ''; position: absolute; z-index: 0; top: 4px; bottom: 4px; left: 4px; width: calc((100% - 8px) / 3); border-radius: 999px; background: var(--surface); box-shadow: 0 2px 8px rgba(20, 25, 60, .1); transition: transform .34s cubic-bezier(.2,.8,.2,1); pointer-events: none; }
.type-tabs.is-langzeit::before { transform: translateX(100%); }
.type-tabs.is-kurzzeit::before { transform: translateX(200%); }
.type-tabs button { position: relative; z-index: 1; border: 0; background: transparent; color: var(--muted); padding: 10px 20px; border-radius: 999px; font-size: 13px; font-weight: 600; cursor: pointer; }
.type-tabs button.active { color: var(--text); }
.results-meta { display: flex; align-items: baseline; gap: 7px; color: var(--muted); font-size: 13px; }
.results-number { color: var(--text); font: 800 28px var(--display); letter-spacing: -1px; }

.listing-grid { position: relative; display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 40px 26px; padding-bottom: 60px; }
.listing-card { min-width: 0; }
.listing-image-wrap { position: relative; aspect-ratio: 4 / 3; overflow: hidden; border-radius: 26px; cursor: pointer; background: var(--surface-soft); }
.listing-image { display: block; width: 100%; height: 100%; object-fit: cover; transition: transform .6s cubic-bezier(.2,.8,.2,1); }
.listing-card:hover .listing-image { transform: scale(1.05); }
.image-shade { position: absolute; inset: 55% 0 0; background: linear-gradient(transparent, rgba(10, 14, 40, .45)); pointer-events: none; }
.listing-type-tag { position: absolute; top: 14px; left: 14px; display: inline-flex; padding: 7px 13px; border-radius: 999px; background: rgba(255, 255, 255, .94); color: #151a33; font-size: 12px; font-weight: 700; }
.favorite-toggle, .detail-favorite { position: absolute; top: 12px; right: 12px; display: grid; place-items: center; width: 40px; height: 40px; border: 0; border-radius: 50%; background: rgba(255, 255, 255, .94); color: #151a33; font-size: 21px; cursor: pointer; }
.favorite-toggle:hover, .detail-favorite:hover { transform: scale(1.08); }
.favorite-toggle.liked, .detail-favorite { color: #ee4d6a; }
.card-delete-btn { position: absolute; top: 12px; right: 60px; display: grid; place-items: center; width: 40px; height: 40px; border: 0; border-radius: 50%; background: rgba(255, 255, 255, .94); color: #d9363e; font-size: 16px; cursor: pointer; }

/* Preis-Badge */
.price-sticker { position: absolute; left: 14px; bottom: 14px; display: flex; align-items: baseline; gap: 5px; padding: 9px 15px; border-radius: 14px; background: var(--sticker); color: var(--sticker-ink); box-shadow: 0 8px 20px rgba(15, 20, 60, .22); transition: transform .25s cubic-bezier(.2,.8,.2,1); line-height: 1; }
.price-sticker strong { font: 800 22px var(--display); letter-spacing: -.6px; white-space: nowrap; }
.price-sticker small { font-size: 11px; font-weight: 700; opacity: .78; }
.listing-card:hover .price-sticker { transform: translateY(-2px); }
.price-sticker.big { padding: 11px 17px; border-radius: 16px; }
.price-sticker.big strong { font-size: 28px; }
.price-sticker.big small { font-size: 12px; }

.listing-content { padding: 16px 4px 0; }
.listing-title { display: block; width: 100%; margin: 0 0 6px; padding: 0; border: 0; background: none; color: var(--text); text-align: left; font: 700 19px var(--display); letter-spacing: -.3px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; cursor: pointer; }
.listing-location { display: flex; align-items: center; gap: 6px; color: var(--muted); font-size: 14px; }
.listing-location > span { color: var(--brand); font-size: 16px; }
.area-pill { margin-left: auto; padding: 3px 10px; border-radius: 999px; background: var(--brand-soft); color: var(--brand); font-size: 12px; font-style: normal; font-weight: 700; }
.listing-card-footer { display: flex; align-items: center; justify-content: space-between; gap: 8px; margin-top: 14px; padding-top: 13px; border-top: 1px dashed var(--line); }
.landlord-mini { display: flex; align-items: center; gap: 8px; min-width: 0; padding: 0; border: 0; background: none; color: var(--muted); font-size: 12px; cursor: pointer; }
.landlord-name { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; color: var(--text); font-weight: 600; }
.listing-age { white-space: nowrap; }
.listing-age::before { content: '·'; margin-right: 8px; }
.own-badge { color: var(--brand); font-weight: 700; }
.mini-avatar { display: grid; place-items: center; width: 26px; height: 26px; flex-shrink: 0; border-radius: 50%; background: var(--brand-soft); color: var(--brand); font-size: 11px; font-weight: 800; }
.details-link { border: 0; background: none; color: var(--brand); font-size: 12px; font-weight: 700; cursor: pointer; white-space: nowrap; }

.state-panel { min-height: 320px; display: flex; flex-direction: column; align-items: center; justify-content: center; gap: 10px; text-align: center; color: var(--muted); }
.state-panel h2 { margin: 8px 0 0; color: var(--text); font: 800 24px var(--display); letter-spacing: -.6px; }
.state-panel p { max-width: 420px; margin: 0 0 8px; font-size: 14px; line-height: 1.6; }
.loading-orbit { width: 34px; height: 34px; border: 3px solid var(--line); border-top-color: var(--brand); border-radius: 50%; animation: spin .8s linear infinite; }
@keyframes spin { to { transform: rotate(360deg); } }
.empty-icon { display: grid; place-items: center; width: 66px; height: 66px; border-radius: 22px; background: var(--sticker); color: var(--sticker-ink); font-size: 32px; }
.secondary-button { display: inline-flex; align-items: center; justify-content: center; min-height: 44px; padding: 10px 20px; border: 1px solid var(--line); border-radius: 999px; background: var(--surface); color: var(--text); font-size: 13px; font-weight: 700; cursor: pointer; }
.site-footer { display: flex; align-items: center; justify-content: space-between; padding: 20px 0 30px; border-top: 1px solid var(--line); color: var(--muted); font-size: 12px; }

/* ---------- Reels ---------- */
.reels-page { width: min(520px, calc(100% - 32px)); margin: 0 auto; padding: 6px 0 20px; }
.reels-heading { display: flex; align-items: center; justify-content: space-between; gap: 12px; padding: 0 4px 12px; }
.back-to-catalog, .reels-filter-button { border: 0; background: none; color: var(--muted); font-size: 13px; font-weight: 700; cursor: pointer; }
.reels-heading-center { display: flex; align-items: center; gap: 7px; color: var(--text); font: 800 15px var(--display); }
.reels-live-dot { width: 8px; height: 8px; border-radius: 50%; background: var(--sticker); box-shadow: 0 0 0 3px color-mix(in srgb, var(--sticker) 35%, transparent); }
.reels-counter { color: var(--muted); font: 500 12px 'Figtree', sans-serif; }
.reels-feed { height: min(760px, calc(100dvh - 150px)); min-height: 460px; overflow-y: auto; scroll-snap-type: y mandatory; scrollbar-width: none; border-radius: 30px; background: #10132b; box-shadow: 0 24px 60px rgba(15, 20, 60, .28); }
.reels-feed::-webkit-scrollbar { display: none; }
.reel-card { position: relative; height: 100%; min-height: 460px; scroll-snap-align: start; scroll-snap-stop: always; overflow: hidden; background: #10132b; color: #fff; }
.reel-image { position: absolute; inset: 0; width: 100%; height: 100%; object-fit: cover; }
.reel-gradient { position: absolute; inset: 48% 0 0; background: linear-gradient(transparent, rgba(8, 10, 30, .88)); pointer-events: none; }
.reel-top-tags { position: absolute; top: 16px; left: 16px; display: flex; align-items: center; gap: 8px; }
.reel-top-tags .listing-type-tag { position: static; }
.reel-date-tag { display: inline-flex; padding: 7px 12px; border-radius: 999px; background: rgba(10, 14, 40, .5); backdrop-filter: blur(10px); color: #fff; font-size: 12px; font-weight: 600; }
.right-floating-bar { position: absolute; right: 14px; bottom: 210px; z-index: 4; display: flex; flex-direction: column; gap: 10px; }
.float-circle { display: grid; place-items: center; width: 46px; height: 46px; border: 0; border-radius: 50%; background: rgba(255, 255, 255, .94); color: #151a33; font-size: 19px; cursor: pointer; transition: transform .2s ease; }
.float-circle:hover { transform: scale(1.08); }
.float-circle.liked { color: #ee4d6a; }
.float-circle.delete-own { color: #d9363e; }

.reel-bottom { position: absolute; z-index: 3; left: 0; right: 0; bottom: 0; display: flex; align-items: flex-end; justify-content: space-between; gap: 16px; padding: 22px 20px 22px; }
.reel-info { min-width: 0; flex: 1; display: flex; flex-direction: column; gap: 8px; text-align: left; }
.reel-location { color: var(--sticker); font-size: 14px; font-weight: 700; }
.reel-title { margin: 0; color: #fff; font: 800 28px/1.08 var(--display); letter-spacing: -1px; display: -webkit-box; -webkit-line-clamp: 3; -webkit-box-orient: vertical; overflow: hidden; text-transform: capitalize; }
.reel-meta-row { display: flex; align-items: center; flex-wrap: wrap; gap: 8px; }
.landlord-chip { display: inline-flex; align-items: center; gap: 7px; padding: 3px 12px 3px 3px; border: 0; border-radius: 999px; background: rgba(255, 255, 255, .16); backdrop-filter: blur(10px); color: #fff; font-size: 12px; font-weight: 600; cursor: pointer; }
.reel-bottom .mini-avatar { width: 24px; height: 24px; background: var(--sticker); color: var(--sticker-ink); }
.own-tag { color: var(--sticker); }
.reel-space-pill { padding: 6px 12px; border-radius: 999px; background: rgba(255, 255, 255, .16); backdrop-filter: blur(10px); color: #fff; font-size: 12px; font-weight: 600; }
.reel-side { display: flex; flex-direction: column; align-items: flex-end; gap: 10px; flex-shrink: 0; }
.reel-side .price-sticker { position: static; }
.reel-details-button { padding: 11px 18px; border: 0; border-radius: 999px; background: #fff; color: #151a33; font-size: 13px; font-weight: 700; cursor: pointer; white-space: nowrap; transition: transform .2s ease; }
.reel-details-button:hover { transform: translateY(-1px); }
.reels-footnote { padding-top: 12px; color: var(--muted); text-align: center; font-size: 12px; }
.reels-empty { min-height: 460px; }
.mobile-bottom-nav { display: none; }

/* ---------- Sheets & Formulare ---------- */
.sheet-backdrop { position: fixed; inset: 0; z-index: 300; display: flex; align-items: flex-end; justify-content: center; padding: 18px; background: rgba(10, 13, 38, .6); backdrop-filter: blur(5px); }
.bottom-sheet { width: min(560px, 100%); max-height: min(90dvh, 850px); overflow-y: auto; padding: 12px 28px 28px; border-radius: 30px; background: var(--surface); color: var(--text); box-shadow: 0 24px 80px rgba(0, 0, 0, .3); }
.drag-bar { width: 44px; height: 5px; margin: 0 auto 20px; border-radius: 9px; background: var(--line); }
.bottom-sheet h3 { margin: 5px 0 8px; font: 800 28px var(--display); letter-spacing: -1px; }
.member-sub { margin: 0 0 18px; color: var(--muted); font-size: 13px; line-height: 1.55; }
.form-label { display: block; margin: 14px 0 7px; color: var(--text); font-size: 13px; font-weight: 700; }
.sheet-input { display: block; width: 100%; min-height: 48px; margin: 0 0 8px; padding: 12px 16px; border: 1px solid var(--line); border-radius: 14px; outline: none; background: var(--page); color: var(--text); font-size: 14px; }
.sheet-input:focus { border-color: var(--brand); box-shadow: 0 0 0 3px color-mix(in srgb, var(--brand) 25%, transparent); }
.sheet-input::placeholder { color: var(--muted); }
.segment-row { display: flex; gap: 7px; margin-bottom: 9px; }
.seg-btn { flex: 1; min-height: 44px; padding: 9px 10px; border: 1px solid var(--line); border-radius: 999px; background: var(--page); color: var(--muted); font-size: 13px; font-weight: 600; cursor: pointer; }
.seg-btn.active { border-color: var(--text); background: var(--text); color: var(--page); }
.two-cols { display: flex; gap: 10px; }
.two-cols > * { flex: 1; min-width: 0; }
.sheet-actions { display: flex; gap: 10px; margin-top: 15px; }
.sheet-actions > * { flex: 1; }
.full-w { width: 100%; margin-top: 12px; }
.error-banner { margin: 10px 0; padding: 11px 14px; border-radius: 12px; background: color-mix(in srgb, var(--danger) 12%, transparent); color: var(--danger); font-size: 13px; font-weight: 700; }
.profile-sheet { text-align: left; }
.profile-top { padding: 6px 0 17px; text-align: center; }
.avatar-circle { display: grid; place-items: center; width: 50px; height: 50px; border-radius: 50%; background: var(--brand-soft); color: var(--brand); font-size: 20px; font-weight: 800; flex-shrink: 0; }
.avatar-circle.large { width: 70px; height: 70px; margin: 0 auto 10px; font-size: 27px; }
.profile-top h2 { margin: 0 0 4px; font: 800 24px var(--display); letter-spacing: -.6px; }
.profile-member { margin: 0; color: var(--muted); font-size: 13px; }
.profile-sheet h4 { margin: 10px 0 12px; font: 700 14px var(--display); }
.mini-list { display: flex; flex-direction: column; gap: 8px; }
.mini-apt-row { display: flex; align-items: center; gap: 12px; width: 100%; padding: 9px; border: 1px solid var(--line); border-radius: 18px; background: var(--page); color: var(--text); text-align: left; cursor: pointer; }
.fav-thumb { width: 54px; height: 54px; flex-shrink: 0; border-radius: 13px; object-fit: cover; }
.mini-apt-info { display: flex; flex: 1; min-width: 0; flex-direction: column; gap: 4px; }
.mini-apt-info strong { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; font-size: 13px; }
.mini-apt-info small { color: var(--muted); font-size: 11px; }
.mini-price { padding: 5px 10px; border-radius: 999px; background: var(--sticker); color: var(--sticker-ink); font-size: 12px; font-weight: 800; white-space: nowrap; }
.account-card { display: flex; align-items: center; justify-content: space-between; gap: 12px; margin: 17px 0; padding: 14px; border-radius: 18px; background: var(--page); }
.account-card-user { display: flex; align-items: center; gap: 10px; min-width: 0; }
.account-card-user > span:last-child { display: flex; flex-direction: column; gap: 4px; min-width: 0; }
.account-card-user strong { font-size: 14px; }
.account-card-user small { color: var(--muted); font-size: 12px; }
.large-avatar { width: 42px; height: 42px; }
.logout-btn { border: 0; border-radius: 999px; padding: 9px 14px; background: color-mix(in srgb, var(--danger) 12%, transparent); color: var(--danger); font-size: 12px; font-weight: 700; cursor: pointer; }
.guest-account { display: flex; align-items: center; justify-content: space-between; width: 100%; color: var(--muted); font-size: 14px; }
.primary-button.compact { padding: 9px 15px; }

/* ---------- Detailseite ---------- */
.detail-overlay { position: fixed; inset: 0; z-index: 220; display: flex; justify-content: center; background: var(--page); color: var(--text); }
.detail-page { display: flex; flex-direction: column; width: min(1180px, 100%); height: 100dvh; background: var(--page); }
.detail-header { display: flex; align-items: center; justify-content: space-between; padding: 18px 28px; }
.detail-header-actions { display: flex; align-items: center; gap: 14px; }
.back-link, .close-link { border: 0; background: none; color: var(--text); font-size: 14px; font-weight: 700; cursor: pointer; }
.close-link { color: var(--brand); }
.delete-text-btn { border: 0; background: color-mix(in srgb, var(--danger) 12%, transparent); color: var(--danger); padding: 8px 14px; border-radius: 999px; font-size: 12px; font-weight: 700; cursor: pointer; }
.detail-scroll { flex: 1; overflow-y: auto; padding: 6px 28px 40px; }
.detail-layout { display: grid; grid-template-columns: minmax(0, 1.15fr) minmax(0, 1fr); gap: 44px; align-items: start; }
.detail-cover-wrap { position: sticky; top: 0; overflow: hidden; height: min(72dvh, 640px); min-height: 260px; border-radius: 32px; background: var(--surface-soft); }
.detail-img { width: 100%; height: 100%; object-fit: cover; }
.detail-type-tag { position: absolute; left: 18px; top: 18px; padding: 8px 14px; border-radius: 999px; background: rgba(255, 255, 255, .94); color: #151a33; font-size: 12px; font-weight: 700; }
.detail-favorite { top: 16px; right: 16px; }
.detail-body { padding: 8px 0 20px; }
.detail-sticker { position: static; display: inline-flex; margin-bottom: 20px; }
.detail-title { margin: 0 0 10px; font: 800 clamp(28px, 3.4vw, 42px)/1.05 var(--display); letter-spacing: -1.4px; text-transform: capitalize; }
.detail-loc { margin: 0; color: var(--muted); font-size: 14px; }
.detail-facts { display: flex; gap: 8px; flex-wrap: wrap; margin-top: 18px; }
.detail-facts span { padding: 8px 14px; border-radius: 999px; background: var(--brand-soft); color: var(--brand); font-size: 12px; font-weight: 700; }
.landlord-profile-card { display: flex; align-items: center; gap: 12px; width: 100%; margin: 26px 0; padding: 14px; border: 1px solid var(--line); border-radius: 20px; background: var(--surface); color: var(--text); text-align: left; cursor: pointer; }
.landlord-info { display: flex; flex: 1; flex-direction: column; gap: 4px; }
.landlord-info strong { font-size: 14px; }
.landlord-info small { color: var(--muted); font-size: 12px; }
.profile-arrow { color: var(--brand); font-size: 18px; }
.detail-body h3 { margin: 0 0 10px; font: 800 20px var(--display); letter-spacing: -.5px; }
.desc-text { margin: 0; max-width: 62ch; color: var(--muted); font-size: 15px; line-height: 1.75; white-space: pre-wrap; }
.detail-bottom-bar { display: flex; gap: 10px; padding: 14px 28px max(16px, env(safe-area-inset-bottom)); border-top: 1px solid var(--line); background: var(--surface); }
.wa-btn, .call-btn { display: flex; flex: 1; align-items: center; justify-content: center; min-height: 52px; border-radius: 999px; text-decoration: none; font-size: 14px; font-weight: 700; }
.wa-btn { background: #c9f2d6; color: #17502f; }
.call-btn { background: var(--brand); color: var(--on-brand); }

/* ---------- Inserat-Formular ---------- */
.listing-form { padding-bottom: max(25px, env(safe-area-inset-bottom)); }
.publisher-line { margin-bottom: 8px; }
.publisher-line strong { color: var(--text); }
.form-grid { align-items: end; }
.form-field { flex: 1; min-width: 0; }
.form-field .form-label { margin-top: 8px; }
.period-segment { gap: 4px; }
.period-segment .seg-btn { padding: 8px 5px; font-size: 12px; }
.upload-box { display: flex; align-items: center; gap: 12px; width: 100%; min-height: 70px; padding: 12px; border: 2px dashed var(--brand); border-radius: 18px; background: var(--brand-soft); color: var(--text); cursor: pointer; }
.upload-box input { display: none; }
.upload-icon { display: grid; place-items: center; width: 42px; height: 42px; border-radius: 13px; background: var(--brand); color: var(--on-brand); font-size: 22px; }
.upload-box > span:nth-child(3) { display: flex; flex: 1; flex-direction: column; gap: 4px; min-width: 0; }
.upload-box strong { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; font-size: 13px; }
.upload-box small { color: var(--muted); font-size: 11px; }
.upload-preview { width: 100%; height: 160px; margin-top: 10px; border-radius: 18px; object-fit: cover; }
.description-input { min-height: 100px; resize: vertical; }
.publish-submit { min-height: 52px; margin-top: 15px; }
.publish-submit:disabled { opacity: .6; cursor: wait; }

/* ---------- Übergänge ---------- */
.view-enter-active, .view-leave-active { transition: opacity .24s ease, transform .28s cubic-bezier(.2,.8,.2,1); }
.view-enter-from { opacity: 0; transform: translateY(10px); }
.view-leave-to { opacity: 0; transform: translateY(-7px); }
.listing-enter-active, .listing-leave-active { transition: opacity .22s ease, transform .24s cubic-bezier(.2,.8,.2,1); }
.listing-enter-from, .listing-leave-to { opacity: 0; transform: translateY(12px) scale(.985); }
.listing-move { transition: transform .28s cubic-bezier(.2,.8,.2,1); }
.listing-leave-active { position: absolute; }
@media (prefers-reduced-motion: reduce) {
  * { transition-duration: .01ms !important; animation-duration: .01ms !important; }
}

/* ---------- Responsive ---------- */
@media (max-width: 980px) {
  .header-inner { width: min(100% - 36px, 1280px); gap: 20px; }
  .nearby-button { display: none; }
  .page-main { width: calc(100% - 36px); }
  .listing-grid { grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 32px 18px; }
  .detail-layout { grid-template-columns: 1fr; gap: 24px; }
  .detail-cover-wrap { position: relative; height: min(56vw, 420px); }
}
@media (max-width: 640px) {
  .app-root { padding-bottom: calc(84px + env(safe-area-inset-bottom)); }
  .header-inner { width: calc(100% - 28px); height: 64px; gap: 10px; }
  .brand-mark { width: 32px; height: 32px; font-size: 11px; }
  .brand-name { font-size: 19px; }
  .desktop-nav, .settings-trigger, .account-label { display: none; }
  .header-actions { gap: 7px; }
  .account-button { padding: 0; }
  .publish-button { padding: 10px 14px; font-size: 12px; }
  .page-main { width: calc(100% - 28px); }
  .search-panel { margin-top: 16px; display: grid; grid-template-columns: 1fr auto; gap: 7px; padding: 8px; border-radius: 26px; }
  .search-field { grid-column: 1 / -1; min-height: 44px; padding: 0 10px; }
  .listing-toolbar { padding: 22px 0 16px; align-items: flex-start; flex-direction: column; gap: 12px; }
  .type-tabs { width: 100%; }
  .listing-grid { grid-template-columns: 1fr; gap: 30px; padding-bottom: 26px; }
  .site-footer { display: none; }
  .mobile-bottom-nav { position: fixed; z-index: 100; left: 14px; right: 14px; bottom: max(12px, env(safe-area-inset-bottom)); display: grid; grid-template-columns: repeat(4, 1fr); gap: 3px; padding: 6px; border-radius: 999px; background: var(--text); box-shadow: 0 10px 30px rgba(10, 14, 40, .3); }
  .mobile-bottom-nav button { position: relative; display: flex; flex-direction: column; align-items: center; justify-content: center; gap: 2px; min-height: 50px; border: 0; border-radius: 999px; background: transparent; color: color-mix(in srgb, var(--page) 60%, transparent); font-size: 10px; font-weight: 600; cursor: pointer; }
  .mobile-bottom-nav button > span { font-size: 19px; line-height: 20px; }
  .mobile-bottom-nav button.active { background: var(--sticker); color: var(--sticker-ink); }
  .mobile-bottom-nav i { position: absolute; top: 3px; right: 18%; display: grid; place-items: center; width: 16px; height: 16px; border-radius: 50%; background: var(--brand); color: var(--on-brand); font-size: 9px; font-style: normal; font-weight: 800; }
  .reels-page { width: 100%; padding: 0; }
  .reels-heading { padding: 10px 16px; }
  .reels-feed { height: calc(100dvh - 150px); min-height: 420px; border-radius: 0; }
  .reel-title { font-size: 24px; }
  .price-sticker.big strong { font-size: 26px; }
  .right-floating-bar { right: 12px; bottom: 190px; }
  .detail-header { padding: 14px 16px; }
  .detail-scroll { padding: 4px 16px 30px; }
  .detail-bottom-bar { padding: 12px 16px max(14px, env(safe-area-inset-bottom)); }
  .sheet-backdrop { padding: 0; align-items: flex-end; }
  .bottom-sheet { width: 100%; max-height: 92dvh; padding: 11px 20px max(24px, env(safe-area-inset-bottom)); border-radius: 28px 28px 0 0; }
}
</style>