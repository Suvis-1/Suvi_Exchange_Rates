<template>
  <div class="container">
    <header class="header">
      <h1>Live Currency Rates (Base: GBP)</h1>
      <p class="timestamp">Last updated: {{ timestamp || 'Loading...' }}</p>
    </header>

    <div class="controls">
      <button @click="isInverse = !isInverse" class="btn">
        {{ isInverse ? 'Normal View (1 GBP = X)' : 'Inverse View (1 CURR = X GBP)' }}
      </button>
      <button @click="isEdit = !isEdit" class="btn secondary">
        {{ isEdit ? 'Done Editing' : 'Edit List' }}
      </button>
    </div>

    <input v-model="searchQuery" placeholder="Search currency (e.g., EUR, BTC)" class="search" />

    <transition name="fade">
      <div v-if="filteredResults.length" class="search-results">
        <div v-for="code in filteredResults" :key="code" class="result-item" @click="addCurrency(code)">
          {{ code }} <span class="add">+ Add</span>
        </div>
      </div>
    </transition>

    <ul class="rates-list" :class="{ 'edit-mode': isEdit }">
      <li v-for="code in displayedCurrencies" :key="code">
        <span class="rate">
          {{ formatRate(code) }}
        </span>
        <span class="remove" @click="removeCurrency(code)">✕ Remove</span>
      </li>
      <li v-if="displayedCurrencies.length === 0" class="empty">No currencies added yet</li>
    </ul>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'

const rates = ref({})
const timestamp = ref('')
const searchQuery = ref('')
const isInverse = ref(false)
const isEdit = ref(false)
const displayedCurrencies = ref(['EUR', 'USD', 'INR', 'JPY', 'BTC', 'CHF', 'CAD', 'AUD', 'CNY'])

const apiUrl = import.meta.env.VITE_API_URL

onMounted(async () => {
  await fetchRates()
  // Update every 1 minute (60 seconds)
  setInterval(fetchRates, 60000)
})

async function fetchRates() {
  try {
    const res = await fetch(apiUrl)
    if (!res.ok) throw new Error('Network error')
    const data = await res.json()
    rates.value = data.rates || {}
    timestamp.value = new Date(data.timestamp).toLocaleString()
  } catch (err) {
    console.error('Fetch error:', err)
    timestamp.value = 'Error loading rates'
  }
}

const allCodes = computed(() => Object.keys(rates.value).sort())

const filteredResults = computed(() => {
  if (!searchQuery.value) return []
  const query = searchQuery.value.toUpperCase()
  return allCodes.value
    .filter(code => code.includes(query) && !displayedCurrencies.value.includes(code))
    .slice(0, 15)
})

function formatRate(code) {
  const rate = rates.value[code]
  if (!rate) return `${code}: N/A`
  
  let value = isInverse ? (1 / rate) : rate
  let formatted = value.toFixed(6).replace(/\.?0+$/, '')
  
  if (isInverse) {
    return `1 ${code} = ${formatted} GBP`
  } else {
    return `1 GBP = ${formatted} ${code}`
  }
}

function addCurrency(code) {
  if (!displayedCurrencies.value.includes(code)) {
    displayedCurrencies.value.push(code)
  }
  searchQuery.value = ''
}

function removeCurrency(code) {
  displayedCurrencies.value = displayedCurrencies.value.filter(c => c !== code)
}
</script>

<style scoped>
* { box-sizing: border-box; }

.container {
  max-width: 900px;
  margin: 0 auto;
  padding: 20px;
  min-height: 100vh;
  background: linear-gradient(to bottom, #f0f4f8, #d9e2ec);
  font-family: 'Segoe UI', Arial, sans-serif;
}

.header {
  text-align: center;
  margin-bottom: 20px;
}

h1 {
  color: #1a3c6d;
  margin-bottom: 8px;
}

.timestamp {
  color: #555;
  font-size: 0.95em;
}

.controls {
  text-align: center;
  margin: 20px 0;
}

.btn {
  padding: 12px 24px;
  margin: 0 10px;
  background: #0066cc;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 1em;
  transition: background 0.3s;
}

.btn:hover {
  background: #0052a3;
}

.btn.secondary {
  background: #666;
}

.btn.secondary:hover {
  background: #444;
}

.search {
  display: block;
  width: 90%;
  max-width: 500px;
  margin: 20px auto;
  padding: 14px;
  font-size: 1.1em;
  border: 1px solid #ccc;
  border-radius: 8px;
  box-shadow: 0 2px 5px rgba(0,0,0,0.1);
}

.search-results {
  max-width: 500px;
  margin: 0 auto 20px;
  background: white;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.15);
  overflow: hidden;
}

.result-item {
  padding: 12px 16px;
  cursor: pointer;
  border-bottom: 1px solid #eee;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.result-item:hover {
  background: #f0f8ff;
}

.add {
  color: #00aa00;
  font-weight: bold;
}

.rates-list {
  list-style: none;
  padding: 0;
  margin: 0 auto;
  max-width: 700px;
}

.rates-list li {
  background: white;
  padding: 18px;
  margin: 12px 0;
  border-radius: 12px;
  box-shadow: 0 4px 10px rgba(0,0,0,0.1);
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 1.2em;
  transition: transform 0.2s;
}

.rates-list li:hover {
  transform: translateY(-2px);
}

.rate {
  font-weight: 600;
  color: #1a3c6d;
}

.remove {
  color: #d00;
  cursor: pointer;
  font-size: 1.1em;
  display: none;
}

.edit-mode .remove {
  display: block;
}

.empty {
  text-align: center;
  color: #888;
  font-style: italic;
}

.fade-enter-active, .fade-leave-active {
  transition: opacity 0.3s;
}

.fade-enter-from, .fade-leave-to {
  opacity: 0;
}
</style>