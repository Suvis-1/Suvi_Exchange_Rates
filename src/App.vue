<template>
  <div class="container">
    <!-- Header -->
    <header class="header">
  <div class="header-top">
    <h1>
      <img src="/SuviExchangeLogo.png" alt="Suvi Exchange Logo" class="currency-logo" />
      Suvi Currency Exchange Rates
    </h1>
    
    <!-- Add vertical separator line -->
    <div class="header-separator"></div>
    
    <div class="header-controls">
      <div class="status-section">
        <div class="status-indicator" :class="{ 'status-error': hasError }">
          <span class="status-dot"></span>
          {{ statusText }}
        </div>
        <div class="auto-refresh-header" v-if="!hasError">
          <label class="refresh-label">
            <input type="checkbox" v-model="autoRefresh" class="refresh-toggle" />
            <span class="refresh-text">Auto-refresh</span>
          </label>
          <div class="refresh-countdown" v-if="autoRefresh">
            {{ formatCountdown }}
          </div>
        </div>
        <p class="timestamp" v-if="timestamp">
          <span class="timestamp-icon">🕒</span>
          {{ timestamp }}
        </p>
      </div>
      <button @click="refreshNow" class="btn-refresh" :disabled="isRefreshing">
        <span class="refresh-icon" :class="{ 'refresh-icon-spinning': isRefreshing }">
          {{ isRefreshing ? '⏳' : '🔄' }}
        </span>
      </button>
    </div>
  </div>
</header>

    <!-- Main Content -->
    <main class="main-content">
      <!-- Search and Controls -->
      <div class="top-controls">
        <div class="search-container">
          <div class="search-wrapper">
            <span class="search-icon">🔍</span>
            <input 
              v-model="searchQuery" 
              placeholder="Search currency (BTC, ETH, USD, Euro, Bitcoin...)" 
              class="search-input"
              @focus="showSearchResults = true"
              @blur="onSearchBlur"
              @mousedown="handleSearchMouseDown"
            />
            <button v-if="searchQuery" @click="clearSearch" class="clear-search" title="Clear search">
              ×
            </button>
          </div>
          
          <!-- Search Results -->
          <transition name="slide-fade">
            <div v-if="showSearchResults && filteredResults.length" class="search-results" @mousedown.prevent>
              <div class="results-scroll">
                <div 
                  v-for="item in filteredResults" 
                  :key="item.code" 
                  class="result-item"
                  @click="addCurrency(item.code)"
                >
                  <div class="result-flag">
                    <img 
                      :src="getFlagUrl(item.code)" 
                      :alt="`${item.code} flag`"
                      class="flag-image"
                      v-if="!isSpecialCurrency(item.code)"
                    />
                    <span v-else class="special-symbol">{{ getSpecialSymbol(item.code) }}</span>
                  </div>
                  <div class="result-details">
                    <strong>{{ item.code }}</strong>
                    <small>{{ item.name }}</small>
                  </div>
                  <span class="add-btn">+</span>
                </div>
              </div>
            </div>
          </transition>
        </div>
        
        <div class="action-buttons">
          <button @click="toggleInverse" class="btn-action" :class="{ 'btn-active': isInverse }" title="Toggle view">
            {{ isInverse ? '1 CUR = X GBP' : '1 GBP = X' }}
          </button>
        </div>
      </div>

      <!-- Currency List -->
      <div class="rates-container">
        <div class="rates-header">
          <div class="rates-title">
            <h2>Tracked Currencies</h2>
            <span class="rates-count">{{ displayedCurrencies.length }}</span>
          </div>
          <div class="view-mode">
            {{ isInverse ? 'Inverse View' : 'Normal View' }}
          </div>
        </div>
        
        <!-- Skeleton loading -->
        <div v-if="loading" class="skeleton-list">
          <div v-for="n in 5" :key="n" class="skeleton-item">
            <div class="skeleton-flag"></div>
            <div class="skeleton-details">
              <div class="skeleton-line skeleton-line-large"></div>
              <div class="skeleton-line skeleton-line-small"></div>
            </div>
            <div class="skeleton-rate"></div>
          </div>
        </div>
        
        <!-- Empty state -->
        <div v-else-if="displayedCurrencies.length === 0" class="empty-state">
          <div class="empty-icon">📊</div>
          <h3>No currencies added</h3>
          <p>Search for currencies above or</p>
        </div>
        
        <!-- Scrollable rates list -->
        <div v-else class="rates-list-scrollable">
          <transition-group name="list" tag="div" class="rates-list">
            <div 
              v-for="code in displayedCurrencies" 
              :key="`${code}-${isInverse}`"
              class="rate-card"
              :class="{ 'rate-card-highlight': hoveredCurrency === code }"
              @mouseenter="hoveredCurrency = code"
              @mouseleave="hoveredCurrency = null"
            >
              <div class="rate-card-content">
                <div class="currency-info">
                  <div class="currency-flag">
                    <img 
                      :src="getFlagUrl(code)" 
                      :alt="`${code} flag`"
                      class="flag-image"
                      v-if="!isSpecialCurrency(code)"
                    />
                    <span v-else class="special-symbol">{{ getSpecialSymbol(code) }}</span>
                  </div>
                  <div class="currency-details">
                    <h3 class="currency-code">{{ code }}</h3>
                    <p class="currency-name">{{ getCurrencyName(code) }}</p>
                  </div>
                </div>
                
                <div class="rate-display">
                  <div class="rate-value">
                    {{ formatRate(code) }}
                  </div>
                  <div class="rate-meta" v-if="getRateChange(code) !== '--'">
                    <span class="rate-trend">{{ getTrendIcon(code) }}</span>
                    <span class="rate-change" :class="getChangeClass(code)">
                      {{ getRateChange(code) }}
                    </span>
                  </div>
                </div>
              </div>
              
              <div class="rate-card-actions">
                <button @click="removeCurrency(code)" class="action-btn" title="Remove">
                  ✕
                </button>
              </div>
            </div>
          </transition-group>
        </div>
      </div>
    </main>
    
    <!-- Toast notifications -->
    <transition name="slide-up">
      <div v-if="toast.show" class="toast" :class="`toast-${toast.type}`">
        <span class="toast-icon">{{ toast.icon }}</span>
        <span class="toast-message">{{ toast.message }}</span>
        <button @click="toast.show = false" class="toast-close">×</button>
      </div>
    </transition>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from 'vue'

// State
const rates = ref({})
const ratesHistory = ref({})
const timestamp = ref('')
const searchQuery = ref('')
const isInverse = ref(false)
const showSearchResults = ref(false)
const hoveredCurrency = ref(null)
const loading = ref(true)
const hasError = ref(false)
const lastUpdateTime = ref(null)
const autoRefresh = ref(true)
const isRefreshing = ref(false)
const countdown = ref(60) // 60 seconds countdown
const toast = ref({
  show: false,
  message: '',
  type: 'info',
  icon: 'ℹ️'
})

// Default currencies
const displayedCurrencies = ref([])

// Currency to country mapping (comprehensive)
const currencyToCountry = {
  AED: 'ae', // UAE
  AFN: 'af', // Afghanistan
  ALL: 'al', // Albania
  AMD: 'am', // Armenia
  ANG: 'sx', // Sint Maarten
  AOA: 'ao', // Angola
  ARS: 'ar', // Argentina
  AUD: 'au', // Australia
  AWG: 'aw', // Aruba
  AZN: 'az', // Azerbaijan
  BAM: 'ba', // Bosnia and Herzegovina
  BBD: 'bb', // Barbados
  BDT: 'bd', // Bangladesh
  BGN: 'bg', // Bulgaria
  BHD: 'bh', // Bahrain
  BIF: 'bi', // Burundi
  BMD: 'bm', // Bermuda
  BND: 'bn', // Brunei
  BOB: 'bo', // Bolivia
  BRL: 'br', // Brazil
  BSD: 'bs', // Bahamas
  BTN: 'bt', // Bhutan
  BWP: 'bw', // Botswana
  BYN: 'by', // Belarus
  BZD: 'bz', // Belize
  CAD: 'ca', // Canada
  CDF: 'cd', // Congo
  CHF: 'ch', // Switzerland
  CLP: 'cl', // Chile
  CNY: 'cn', // China
  COP: 'co', // Colombia
  CRC: 'cr', // Costa Rica
  CUP: 'cu', // Cuba
  CVE: 'cv', // Cape Verde
  CZK: 'cz', // Czech Republic
  DJF: 'dj', // Djibouti
  DKK: 'dk', // Denmark
  DOP: 'do', // Dominican Republic
  DZD: 'dz', // Algeria
  EGP: 'eg', // Egypt
  ERN: 'er', // Eritrea
  ETB: 'et', // Ethiopia
  FJD: 'fj', // Fiji
  FKP: 'fk', // Falkland Islands
  GBP: 'gb', // United Kingdom
  GEL: 'ge', // Georgia
  GHS: 'gh', // Ghana
  GIP: 'gi', // Gibraltar
  GMD: 'gm', // Gambia
  GNF: 'gn', // Guinea
  GTQ: 'gt', // Guatemala
  GYD: 'gy', // Guyana
  HKD: 'hk', // Hong Kong
  HNL: 'hn', // Honduras
  HRK: 'hr', // Croatia
  HTG: 'ht', // Haiti
  HUF: 'hu', // Hungary
  IDR: 'id', // Indonesia
  ILS: 'il', // Israel
  INR: 'in', // India
  IQD: 'iq', // Iraq
  IRR: 'ir', // Iran
  ISK: 'is', // Iceland
  JMD: 'jm', // Jamaica
  JOD: 'jo', // Jordan
  JPY: 'jp', // Japan
  KES: 'ke', // Kenya
  KGS: 'kg', // Kyrgyzstan
  KHR: 'kh', // Cambodia
  KMF: 'km', // Comoros
  KPW: 'kp', // North Korea
  KRW: 'kr', // South Korea
  KWD: 'kw', // Kuwait
  KYD: 'ky', // Cayman Islands
  KZT: 'kz', // Kazakhstan
  LAK: 'la', // Laos
  LBP: 'lb', // Lebanon
  LKR: 'lk', // Sri Lanka
  LRD: 'lr', // Liberia
  LSL: 'ls', // Lesotho
  LYD: 'ly', // Libya
  MAD: 'ma', // Morocco
  MDL: 'md', // Moldova
  MGA: 'mg', // Madagascar
  MKD: 'mk', // North Macedonia
  MMK: 'mm', // Myanmar
  MNT: 'mn', // Mongolia
  MOP: 'mo', // Macau
  MRU: 'mr', // Mauritania
  MUR: 'mu', // Mauritius
  MVR: 'mv', // Maldives
  MWK: 'mw', // Malawi
  MXN: 'mx', // Mexico
  MYR: 'my', // Malaysia
  MZN: 'mz', // Mozambique
  NAD: 'na', // Namibia
  NGN: 'ng', // Nigeria
  NIO: 'ni', // Nicaragua
  NOK: 'no', // Norway
  NPR: 'np', // Nepal
  NZD: 'nz', // New Zealand
  OMR: 'om', // Oman
  PAB: 'pa', // Panama
  PEN: 'pe', // Peru
  PGK: 'pg', // Papua New Guinea
  PHP: 'ph', // Philippines
  PKR: 'pk', // Pakistan
  PLN: 'pl', // Poland
  PYG: 'py', // Paraguay
  QAR: 'qa', // Qatar
  RON: 'ro', // Romania
  RSD: 'rs', // Serbia
  RUB: 'ru', // Russia
  RWF: 'rw', // Rwanda
  SAR: 'sa', // Saudi Arabia
  SBD: 'sb', // Solomon Islands
  SCR: 'sc', // Seychelles
  SDG: 'sd', // Sudan
  SEK: 'se', // Sweden
  SGD: 'sg', // Singapore
  SHP: 'sh', // Saint Helena
  SLL: 'sl', // Sierra Leone
  SOS: 'so', // Somalia
  SRD: 'sr', // Suriname
  SSP: 'ss', // South Sudan
  STN: 'st', // São Tomé and Príncipe
  SYP: 'sy', // Syria
  SZL: 'sz', // Eswatini
  THB: 'th', // Thailand
  TJS: 'tj', // Tajikistan
  TMT: 'tm', // Turkmenistan
  TND: 'tn', // Tunisia
  TOP: 'to', // Tonga
  TRY: 'tr', // Turkey
  TTD: 'tt', // Trinidad and Tobago
  TWD: 'tw', // Taiwan
  TZS: 'tz', // Tanzania
  UAH: 'ua', // Ukraine
  UGX: 'ug', // Uganda
  USD: 'us', // United States
  UYU: 'uy', // Uruguay
  UZS: 'uz', // Uzbekistan
  VES: 've', // Venezuela
  VND: 'vn', // Vietnam
  VUV: 'vu', // Vanuatu
  WST: 'ws', // Samoa
  YER: 'ye', // Yemen
  ZAR: 'za', // South Africa
  ZMW: 'zm', // Zambia
  ZWL: 'zw', // Zimbabwe
}

// Special currencies with symbols
const specialCurrencies = {
  // Cryptocurrencies
  BTC: { symbol: '₿', name: 'Bitcoin' },
  ETH: { symbol: 'Ξ', name: 'Ethereum' },
  ADA: { symbol: '₳', name: 'Cardano' },
  BNB: { symbol: 'ⓑ', name: 'Binance Coin' },
  BCH: { symbol: 'Ƀ', name: 'Bitcoin Cash' },
  DOGE: { symbol: 'Ð', name: 'Dogecoin' },
  DOT: { symbol: '●', name: 'Polkadot' },
  LINK: { symbol: '🔗', name: 'Chainlink' },
  LTC: { symbol: 'Ł', name: 'Litecoin' },
  SOL: { symbol: '◎', name: 'Solana' },
  UNI: { symbol: '🦄', name: 'Uniswap' },
  XRP: { symbol: '✕', name: 'Ripple' },
  XLM: { symbol: '*', name: 'Stellar' },
  LUNA: { symbol: '🌙', name: 'Terra' },
  XBT: { symbol: '₿', name: 'Bitcoin (XBT)' },
  
  // Stablecoins
  USDC: { symbol: '💵', name: 'USD Coin' },
  USDT: { symbol: '💵', name: 'Tether' },
  USDP: { symbol: '💵', name: 'Pax Dollar' },
  EURC: { symbol: '💶', name: 'Euro Coin' },
  
  // Special currencies
  EUR: { symbol: '€', name: 'Euro', flag: 'eu' },
  XAF: { symbol: 'FCFA', name: 'Central African CFA Franc', flag: 'cm' },
  XOF: { symbol: 'CFA', name: 'West African CFA Franc', flag: 'sn' },
  XCD: { symbol: '$', name: 'East Caribbean Dollar', flag: 'ag' },
  XDR: { symbol: 'SDR', name: 'Special Drawing Rights', flag: 'imf' },
  XAU: { symbol: '🥇', name: 'Gold (ounce)' },
  XAG: { symbol: '🥈', name: 'Silver (ounce)' },
  XPT: { symbol: '⚙️', name: 'Platinum' },
  XPD: { symbol: '🔩', name: 'Palladium' },


  // Major fiat currencies
  USD: { symbol: '$', name: 'US Dollar', flag: 'us' },
  JPY: { symbol: '¥', name: 'Japanese Yen', flag: 'jp' },
  GBP: { symbol: '£', name: 'British Pound', flag: 'gb' },
  AUD: { symbol: '$', name: 'Australian Dollar', flag: 'au' },
  CAD: { symbol: '$', name: 'Canadian Dollar', flag: 'ca' },
  CHF: { symbol: 'Fr', name: 'Swiss Franc', flag: 'ch' },
  CNY: { symbol: '¥', name: 'Chinese Yuan', flag: 'cn' },
  INR: { symbol: '₹', name: 'Indian Rupee', flag: 'in' },
  BRL: { symbol: 'R$', name: 'Brazilian Real', flag: 'br' },
  RUB: { symbol: '₽', name: 'Russian Ruble', flag: 'ru' },
  KRW: { symbol: '₩', name: 'South Korean Won', flag: 'kr' },
  MXN: { symbol: '$', name: 'Mexican Peso', flag: 'mx' },
  ZAR: { symbol: 'R', name: 'South African Rand', flag: 'za' },
  
  // Historical currencies
  ATS: { symbol: 'ÖS', name: 'Austrian Schilling' },
  BEF: { symbol: 'FB', name: 'Belgian Franc' },
  CYP: { symbol: '£', name: 'Cypriot Pound' },
  DEM: { symbol: 'DM', name: 'German Mark' },
  ESP: { symbol: '₧', name: 'Spanish Peseta' },
  FIM: { symbol: 'mk', name: 'Finnish Markka' },
  FRF: { symbol: '₣', name: 'French Franc' },
  GRD: { symbol: 'Δρ', name: 'Greek Drachma' },
  IEP: { symbol: '£', name: 'Irish Pound' },
  ITL: { symbol: '₤', name: 'Italian Lira' },
  LUF: { symbol: 'F', name: 'Luxembourg Franc' },
  MTL: { symbol: '₤', name: 'Maltese Lira' },
  NLG: { symbol: 'ƒ', name: 'Dutch Guilder' },
  PTE: { symbol: 'Esc', name: 'Portuguese Escudo' },
  SIT: { symbol: 'SIT', name: 'Slovenian Tolar' },
  SKK: { symbol: 'Sk', name: 'Slovak Koruna' },
  VAL: { symbol: 'V', name: 'Vale' },
  
  // Obsolete currencies
  EEK: { symbol: 'kr', name: 'Estonian Kroon' },
  LTL: { symbol: 'Lt', name: 'Lithuanian Litas' },
  LVL: { symbol: 'Ls', name: 'Latvian Lats' },
  MGF: { symbol: 'MG', name: 'Malagasy Franc' },
  MRO: { symbol: 'UM', name: 'Mauritanian Ouguiya' },
  MZM: { symbol: 'Mt', name: 'Mozambican Metical' },
  SDD: { symbol: 'LSd', name: 'Sudanese Dinar' },
  SRG: { symbol: 'ƒ', name: 'Surinamese Guilder' },
  STD: { symbol: 'Db', name: 'São Tomé Dobra' },
  TRL: { symbol: '₤', name: 'Turkish Lira (old)' },
  VEB: { symbol: 'Bs', name: 'Venezuelan Bolívar (old)' },
  VEF: { symbol: 'Bs.F', name: 'Venezuelan Bolívar Fuerte' },
  ZMK: { symbol: 'ZK', name: 'Zambian Kwacha (old)' },
  ZWD: { symbol: 'Z$', name: 'Zimbabwean Dollar (old)' },
  ZWG: { symbol: 'Z$', name: 'Zimbabwean Gold' },
}

// Function to generate currency name for unknown currencies
function generateCurrencyName(code) {
  // Check if it's already in specialCurrencies
  if (specialCurrencies[code]) {
    return specialCurrencies[code].name
  }
  
  // Comprehensive currency names database
  const currencyNames = {
    // Major currencies
    'USD': 'US Dollar',
    'EUR': 'Euro',
    'JPY': 'Japanese Yen',
    'GBP': 'British Pound',
    'AUD': 'Australian Dollar',
    'CAD': 'Canadian Dollar',
    'CHF': 'Swiss Franc',
    'CNY': 'Chinese Yuan',
    'HKD': 'Hong Kong Dollar',
    'NZD': 'New Zealand Dollar',
    
    // Asian currencies
    'INR': 'Indian Rupee',
    'KRW': 'South Korean Won',
    'SGD': 'Singapore Dollar',
    'THB': 'Thai Baht',
    'TWD': 'New Taiwan Dollar',
    'IDR': 'Indonesian Rupiah',
    'MYR': 'Malaysian Ringgit',
    'PHP': 'Philippine Peso',
    'VND': 'Vietnamese Dong',
    'PKR': 'Pakistani Rupee',
    'BDT': 'Bangladeshi Taka',
    'LKR': 'Sri Lankan Rupee',
    'NPR': 'Nepalese Rupee',
    'MMK': 'Burmese Kyat',
    'KHR': 'Cambodian Riel',
    'LAK': 'Lao Kip',
    'MNT': 'Mongolian Tugrik',
    
    // Middle Eastern currencies
    'AED': 'UAE Dirham',
    'SAR': 'Saudi Riyal',
    'QAR': 'Qatari Riyal',
    'OMR': 'Omani Rial',
    'KWD': 'Kuwaiti Dinar',
    'BHD': 'Bahraini Dinar',
    'JOD': 'Jordanian Dinar',
    'ILS': 'Israeli New Shekel',
    'IRR': 'Iranian Rial',
    'IQD': 'Iraqi Dinar',
    'YER': 'Yemeni Rial',
    'SYP': 'Syrian Pound',
    'LBP': 'Lebanese Pound',
    
    // European currencies
    'SEK': 'Swedish Krona',
    'NOK': 'Norwegian Krone',
    'DKK': 'Danish Krone',
    'PLN': 'Polish Złoty',
    'CZK': 'Czech Koruna',
    'HUF': 'Hungarian Forint',
    'RON': 'Romanian Leu',
    'BGN': 'Bulgarian Lev',
    'HRK': 'Croatian Kuna',
    'RSD': 'Serbian Dinar',
    'ALL': 'Albanian Lek',
    'MKD': 'Macedonian Denar',
    'BAM': 'Bosnia-Herzegovina Convertible Mark',
    'GEL': 'Georgian Lari',
    'AMD': 'Armenian Dram',
    'AZN': 'Azerbaijani Manat',
    
    // African currencies
    'ZAR': 'South African Rand',
    'EGP': 'Egyptian Pound',
    'NGN': 'Nigerian Naira',
    'KES': 'Kenyan Shilling',
    'ETB': 'Ethiopian Birr',
    'GHS': 'Ghanaian Cedi',
    'TZS': 'Tanzanian Shilling',
    'UGX': 'Ugandan Shilling',
    'MAD': 'Moroccan Dirham',
    'DZD': 'Algerian Dinar',
    'SDG': 'Sudanese Pound',
    'TND': 'Tunisian Dinar',
    'LYD': 'Libyan Dinar',
    'XOF': 'West African CFA Franc',
    'XAF': 'Central African CFA Franc',
    'CDF': 'Congolese Franc',
    'RWF': 'Rwandan Franc',
    'BIF': 'Burundian Franc',
    'DJF': 'Djiboutian Franc',
    'SOS': 'Somali Shilling',
    'MZN': 'Mozambican Metical',
    'ZMW': 'Zambian Kwacha',
    'MWK': 'Malawian Kwacha',
    'AOA': 'Angolan Kwanza',
    'MUR': 'Mauritian Rupee',
    'SCR': 'Seychellois Rupee',
    'MVR': 'Maldivian Rufiyaa',
    
    // Americas currencies
    'BRL': 'Brazilian Real',
    'MXN': 'Mexican Peso',
    'ARS': 'Argentine Peso',
    'CLP': 'Chilean Peso',
    'COP': 'Colombian Peso',
    'PEN': 'Peruvian Sol',
    'UYU': 'Uruguayan Peso',
    'PYG': 'Paraguayan Guaraní',
    'BOB': 'Bolivian Boliviano',
    'GTQ': 'Guatemalan Quetzal',
    'CRC': 'Costa Rican Colón',
    'PAB': 'Panamanian Balboa',
    'DOP': 'Dominican Peso',
    'HTG': 'Haitian Gourde',
    'JMD': 'Jamaican Dollar',
    'TTD': 'Trinidad and Tobago Dollar',
    'BSD': 'Bahamian Dollar',
    'BBD': 'Barbadian Dollar',
    'XCD': 'East Caribbean Dollar',
    'BZD': 'Belize Dollar',
    
    // Cryptocurrencies (already in specialCurrencies, but added for completeness)
    'BTC': 'Bitcoin',
    'ETH': 'Ethereum',
    'BNB': 'Binance Coin',
    'ADA': 'Cardano',
    'SOL': 'Solana',
    'XRP': 'Ripple',
    'DOT': 'Polkadot',
    'DOGE': 'Dogecoin',
    'LTC': 'Litecoin',
    'BCH': 'Bitcoin Cash',
    'LINK': 'Chainlink',
    'XLM': 'Stellar',
    'UNI': 'Uniswap',
    'USDT': 'Tether',
    'USDC': 'USD Coin',
    
    // Other special currencies
    'XAU': 'Gold (ounce)',
    'XAG': 'Silver (ounce)',
    'XPT': 'Platinum',
    'XPD': 'Palladium',
    'XDR': 'Special Drawing Rights',
  }
  
  if (currencyNames[code]) {
    return currencyNames[code]
  }
  
  // If not found in our database, check patterns
  if (/^[A-Z]{3,4}$/.test(code)) {
    // Could be a lesser-known cryptocurrency
    const cryptoPatterns = {
      'BTC': 'Bitcoin',
      'ETH': 'Ethereum', 
      'XRP': 'Ripple',
      'LTC': 'Litecoin',
      'BCH': 'Bitcoin Cash',
      'ADA': 'Cardano',
      'DOT': 'Polkadot',
      'LINK': 'Chainlink',
      'BNB': 'Binance Coin',
      'XLM': 'Stellar',
      'DOGE': 'Dogecoin',
      'SOL': 'Solana',
      'LUNA': 'Terra',
      'UNI': 'Uniswap',
      'XBT': 'Bitcoin',
      'USDC': 'USD Coin',
      'USDT': 'Tether',
      'USDP': 'Pax Dollar',
      'EURC': 'Euro Coin',
    }
    
    if (cryptoPatterns[code]) {
      return cryptoPatterns[code]
    }
    
    // Generic name for unknown 3-4 letter codes
    return `${code}`
  }
  
  // Default fallback
  return `${code} Currency`
}


// Currency metadata with names - dynamically generated
const currencyMetadata = computed(() => {
  const metadata = {}
  
  // Add all currencies from rates
  Object.keys(rates.value).forEach(code => {
    if (code !== 'GBP') {
      // Check special currencies first
      if (specialCurrencies[code]) {
        metadata[code] = { name: specialCurrencies[code].name }
      } else {
        // Generate name based on pattern
        metadata[code] = { name: generateCurrencyName(code) }
      }
    }
  })
  
  return metadata
})

//this function to create a search index 
function createCurrencySearchIndex(code) {
  const meta = currencyMetadata.value[code]
  if (!meta) return []
  
  const searchTerms = [code.toLowerCase(), meta.name.toLowerCase()]
  
  // Add common aliases based on currency type
  const aliases = {
    'INR': ['rupee', 'india', 'indian', 'indian rupee', '₹'],
    'USD': ['dollar', 'usd', 'us', 'us dollar', '$'],
    'EUR': ['euro', 'europe', 'european', '€'],
    'GBP': ['pound', 'sterling', 'british', 'british pound', 'uk', '£'],
    'JPY': ['yen', 'japan', 'japanese', 'japanese yen', '¥'],
    'CNY': ['yuan', 'china', 'chinese', 'chinese yuan', 'rmb'],
    'RUB': ['ruble', 'russia', 'russian', 'russian ruble', '₽'],
    'KRW': ['won', 'korea', 'korean', 'south korean won', '₩'],
    'THB': ['baht', 'thailand', 'thai', 'thai baht'],
    'IDR': ['rupiah', 'indonesia', 'indonesian'],
    'PHP': ['peso', 'philippines', 'philippine'],
    'MYR': ['ringgit', 'malaysia', 'malaysian'],
    'VND': ['dong', 'vietnam', 'vietnamese'],
    'PKR': ['pakistan rupee', 'pakistani rupee'],
    'LKR': ['sri lanka rupee', 'sri lankan rupee'],
    'NPR': ['nepal rupee', 'nepalese rupee'],
    'MVR': ['rufiyaa', 'maldives', 'maldivian'],
    'SCR': ['seychelles rupee'],
    'MUR': ['mauritius rupee'],
  }
  
  if (aliases[code]) {
    searchTerms.push(...aliases[code])
  }
  
  return searchTerms
}

// Computed properties
const allCodes = computed(() => {
  return Object.keys(rates.value)
    .filter(code => code !== 'GBP')
    .sort((a, b) => a.localeCompare(b))
})

const filteredResults = computed(() => {
  if (!searchQuery.value) return []
  
  const query = searchQuery.value.toLowerCase().trim()
  const addedCodes = new Set(displayedCurrencies.value)
  
  // If query is very short (1-2 chars), only search by code
  if (query.length <= 2) {
    return allCodes.value
      .filter(code => {
        if (addedCodes.has(code)) return false
        return code.toLowerCase().includes(query)
      })
      .slice(0, 8)
      .map(code => ({
        code,
        name: currencyMetadata.value[code]?.name || generateCurrencyName(code),
      }))
  }
  
  // For longer queries, use full search
  return allCodes.value
    .filter(code => {
      if (addedCodes.has(code)) return false
      
      const searchTerms = createCurrencySearchIndex(code)
      return searchTerms.some(term => term.includes(query))
    })
    .slice(0, 12)
    .map(code => ({
      code,
      name: currencyMetadata.value[code]?.name || generateCurrencyName(code),
    }))
})

const statusText = computed(() => {
  if (hasError.value) return 'Error'
  if (loading.value) return 'Loading...'
  if (!lastUpdateTime.value) return 'Waiting'
  return 'Live'
})

// Add this missing computed property:
const formatCountdown = computed(() => {
  if (countdown.value <= 0) return '0s'
  return `${countdown.value}s`
})

// Lifecycle
let refreshInterval = null
let countdownInterval = null

onMounted(async () => {
  loadUserPreferences()
  await fetchRates()
  startCountdown()
})

onUnmounted(() => {
  if (refreshInterval) clearInterval(refreshInterval)
  if (countdownInterval) clearInterval(countdownInterval)
  saveUserPreferences()
})

// Watch for changes
watch(autoRefresh, (newValue) => {
  if (newValue) {
    startCountdown()
  } else {
    stopCountdown()
  }
})

watch(displayedCurrencies, () => {
  saveUserPreferences()
}, { deep: true })

// Methods
function startCountdown() {
  if (countdownInterval) clearInterval(countdownInterval)
  countdown.value = 60
  
  countdownInterval = setInterval(() => {
    countdown.value--
    
    if (countdown.value <= 0) {
      if (autoRefresh.value && document.visibilityState === 'visible') {
        fetchRates()
      }
      countdown.value = 60 // Reset countdown
    }
  }, 1000)
}

function stopCountdown() {
  if (countdownInterval) clearInterval(countdownInterval)
  countdown.value = 60
}

async function fetchRates() {
  if (isRefreshing.value) return
  
  isRefreshing.value = true
  loading.value = true
  
  try {
   const apiUrl = import.meta.env.VITE_API_URL 
    
    let data = null
    let lastError = null
    

      const res = await fetch(apiUrl)
      if (res.ok) {
        data = await res.json()
      } else {
        throw new Error(`API returned ${res.status}`)
      }

        
    rates.value = data.rates || {}
    
    // Store rate history for change calculations
    const now = new Date()
    Object.keys(rates.value).forEach(code => {
      if (!ratesHistory.value[code]) {
        ratesHistory.value[code] = []
      }
      ratesHistory.value[code].push({
        rate: rates.value[code],
        timestamp: now
      })
      
      // Keep only last 10 entries
      if (ratesHistory.value[code].length > 10) {
        ratesHistory.value[code].shift()
      }
    })
    
    // Update timestamp
    if (data.timestamp) {
      // Convert MongoDB timestamp to Date
      const ts = data.timestamp
      timestamp.value = new Date(ts).toLocaleString('en-GB', {
        day: '2-digit',
        month: 'short',
        hour: '2-digit',
        minute: '2-digit',
        second: '2-digit'
      })
    } else {
      timestamp.value = new Date().toLocaleString('en-GB', {
        day: '2-digit',
        month: 'short',
        hour: '2-digit',
        minute: '2-digit',
        second: '2-digit'
      })
    }
    
    lastUpdateTime.value = new Date()
    
    if (!hasError.value) {
      showToast('Live rates updated', 'success', '✅')
    }
  } catch (err) {
    console.error('Fetch error:', err)
    hasError.value = true
    showToast('Using offline data', 'error', '⚠️')
  } finally {
    loading.value = false
    isRefreshing.value = false
  }
}


async function refreshNow() {
  await fetchRates()
  if (autoRefresh.value) {
    startCountdown()
  }
}

function toggleInverse() {
  isInverse.value = !isInverse.value
  saveUserPreferences()
}

function formatRate(code) {
  const rate = rates.value[code]
  if (!rate || rate === 0) return '...'
  
  let value = isInverse.value ? (1 / rate) : rate
  
  // For very small values (less than 0.001), always show the actual value
  if (Math.abs(value) < 0.001) {
    // Show the actual number without rounding to zero
    const fixedValue = value.toFixed(10) // Use 10 decimal places to capture small values
    // Remove trailing zeros
    const trimmed = fixedValue.replace(/(\.\d*?[1-9])0+$/, '$1').replace(/\.0+$/, '')
    
    // If still very small and might not show enough digits
    if (trimmed === '0' || trimmed === '0.0' || trimmed === '0.00') {
      // Force showing at least 8 decimal places for extremely small values
      return value.toFixed(8).replace(/(\.\d*?[1-9])0+$/, '$1')
    }
    
    return trimmed
  }
  
  // For regular values, show 5 total digits
  return formatToFiveDigits(value)
}

function formatToFiveDigits(num) {
  // Convert to string to check
  const str = num.toString()
  
  // If in scientific notation, convert to regular
  if (str.includes('e')) {
    const fixed = num.toFixed(20).replace(/\.?0+$/, '')
    return formatToFiveDigits(parseFloat(fixed))
  }
  
  // Get integer part length
  const integerPart = Math.floor(num)
  const integerDigits = integerPart.toString().length
  
  if (integerDigits >= 5) {
    // 5+ integer digits: show as integer
    return integerPart.toLocaleString('en-US')
  } else if (integerDigits >= 1) {
    // 1-4 integer digits: show them + decimal digits to make 5 total
    const decimalDigitsNeeded = 5 - integerDigits
    return num.toLocaleString('en-US', {
      minimumFractionDigits: decimalDigitsNeeded,
      maximumFractionDigits: decimalDigitsNeeded
    })
  } else {
    // Less than 1: always show 5 decimal places
    return num.toLocaleString('en-US', {
      minimumFractionDigits: 5,
      maximumFractionDigits: 5
    })
  }
}

function getCurrencyName(code) {
  if (specialCurrencies[code]) {
    return specialCurrencies[code].name
  }
  return currencyMetadata.value[code]?.name || generateCurrencyName(code)
}

// Helper functions for flags
function isSpecialCurrency(code) {
  return code in specialCurrencies && !specialCurrencies[code].flag
}

function getSpecialSymbol(code) {
  return specialCurrencies[code]?.symbol || code
}

function getFlagUrl(code) {
  // Check for special currencies with flags
  if (specialCurrencies[code]?.flag) {
    return `https://flagcdn.com/w40/${specialCurrencies[code].flag}.png`
  }
  
  // Check for regular country-based currencies
  const country = currencyToCountry[code]
  if (country) {
    return `https://flagcdn.com/w40/${country}.png`
  }
  
  // For cryptocurrencies and special currencies without flags, return null to use symbol
  if (specialCurrencies[code]) {
    return null
  }
  
  // Fallback: use a generic flag
  return 'https://flagcdn.com/w40/un.png'
}

function getRateChange(code) {
  const history = ratesHistory.value[code]
  if (!history || history.length < 2) return '--'
  
  const current = history[history.length - 1].rate
  const previous = history[history.length - 2].rate
  const change = ((current - previous) / previous * 100)
  
  if (Math.abs(change) < 0.01) return '0.00%'
  return change > 0 ? `+${change.toFixed(2)}%` : `${change.toFixed(2)}%`
}

function getTrendIcon(code) {
  const change = getRateChange(code)
  if (change.includes('+')) return '↗'
  if (change.includes('-') && !change.includes('--')) return '↘'
  return '→'
}

function getChangeClass(code) {
  const change = getRateChange(code)
  if (change.includes('+')) return 'change-positive'
  if (change.includes('-') && !change.includes('--')) return 'change-negative'
  return 'change-neutral'
}

function addCurrency(code) {
  if (!displayedCurrencies.value.includes(code)) {
    displayedCurrencies.value.push(code)
    showToast(`${getCurrencyName(code)} added`, 'success', '✅')
  }
  searchQuery.value = ''
  // Don't hide results immediately - let user continue searching
  // showSearchResults.value = false
}

function removeCurrency(code) {
  displayedCurrencies.value = displayedCurrencies.value.filter(c => c !== code)
  showToast(`${getCurrencyName(code)} removed`, 'info', '🗑️')
}

function clearSearch() {
  searchQuery.value = ''
  showSearchResults.value = false
}

function onSearchBlur() {
  // Use a longer delay to ensure click events on results are processed
  setTimeout(() => {
    showSearchResults.value = false
  }, 300) // Increased from 200ms to 300ms
}

function handleSearchMouseDown() {
  // If search results are hidden but we have a query, show them
  if (searchQuery.value && !showSearchResults.value) {
    showSearchResults.value = true
  }
}

function showToast(message, type = 'info', icon = 'ℹ️') {
  toast.value = {
    show: true,
    message,
    type,
    icon
  }
  
  setTimeout(() => {
    toast.value.show = false
  }, 3000)
}

function loadUserPreferences() {
  try {
    const saved = localStorage.getItem('currencyDashboardPrefs')
    if (saved) {
      const prefs = JSON.parse(saved)
      if (prefs.currencies && Array.isArray(prefs.currencies) && prefs.currencies.length > 0) {
        displayedCurrencies.value = prefs.currencies
      } else {
        displayedCurrencies.value = ['BTC', 'ETH', 'USD', 'EUR', 'JPY', 'ADA', 'SOL', 'XRP']
      }
      if (prefs.isInverse !== undefined) isInverse.value = prefs.isInverse
      if (prefs.autoRefresh !== undefined) autoRefresh.value = prefs.autoRefresh
    } else {
      displayedCurrencies.value = ['BTC', 'ETH', 'USD', 'EUR', 'JPY', 'ADA', 'SOL', 'XRP']
    }
  } catch (err) {
    console.error('Failed to load preferences:', err)
    displayedCurrencies.value = ['BTC', 'ETH', 'USD', 'EUR', 'JPY', 'ADA', 'SOL', 'XRP']
  }
}

function saveUserPreferences() {
  try {
    const prefs = {
      currencies: displayedCurrencies.value,
      isInverse: isInverse.value,
      autoRefresh: autoRefresh.value,
      lastSaved: new Date().toISOString()
    }
    localStorage.setItem('currencyDashboardPrefs', JSON.stringify(prefs))
  } catch (err) {
    console.error('Failed to save preferences:', err)
  }
}
</script>

<style scoped>
* { 
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

.container {
  max-width: 100%;
  height: 100vh;
  padding: 12px;
  background: linear-gradient(135deg, #035eb9 0%, #035eb9 100%);
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  color: #334155;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

/* Header */
.header {
  background: white;
  border-radius: 12px;
  padding: 16px;
  margin-bottom: 12px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  flex-shrink: 0;
}

.header-top {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0; /* Remove bottom margin */
  gap: 20px; /* Add gap between elements */
}

h1 {
  color: #1e293b;
  font-size: 2.5rem; /* Slightly reduced from 3rem */
  font-weight: 700;
  display: flex;
  align-items: center;
  gap: 8px;
  flex: 1; /* Allow it to take available space */
}

.currency-logo {
  width: 150px; 
  height: 150px; 
  object-fit: contain;
  margin-right: 8px;
}

.header-controls {
  display: flex;
  align-items: center;
  gap: 16px;
}

/* Vertical separator line */
.header-separator {
  width: 1px;
  height: 60px; /* Adjust height as needed */
  background-color: #e2e8f0; /* Light gray color */
  margin: 0 20px; /* Space on both sides */
}

.header-controls {
  display: flex;
  align-items: center;
  gap: 20px; /* Increased gap */
  flex-shrink: 0; /* Prevent shrinking */
}

/* Status section - make it neat */
.status-section {
  display: flex;
  flex-direction: column;
  gap: 8px; /* Slightly more gap */
  align-items: flex-end;
  min-width: 200px;
  background: #f8fafc; /* Light background */
  padding: 12px 16px;
  border-radius: 10px;
  border: 1px solid #e2e8f0; /* Border for neat appearance */
}

/* Status indicator */
.status-indicator {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 12px;
  background: #dcfce7;
  border-radius: 8px;
  font-size: 0.8rem;
  font-weight: 500;
  color: #166534;
  width: 100%;
  justify-content: center;
}

.status-error {
  background: #fee2e2;
  color: #991b1b;
}

.status-dot {
  width: 8px;
  height: 8px;
  background: #22c55e;
  border-radius: 50%;
  animation: pulse 2s infinite;
}

.status-error .status-dot {
  background: #ef4444;
  animation: none;
}

/* Auto-refresh header */
.auto-refresh-header {
  display: flex;
  align-items: center;
  gap: 10px;
  font-size: 0.8rem;
  width: 100%;
  justify-content: center;
  padding: 6px 0;
  border-top: 1px solid #e2e8f0; /* Separator line */
  border-bottom: 1px solid #e2e8f0; /* Separator line */
}

.refresh-label {
  display: flex;
  align-items: center;
  gap: 6px;
  cursor: pointer;
  user-select: none;
  color: #475569;
}

.refresh-toggle {
  width: 16px;
  height: 16px;
  cursor: pointer;
  accent-color: #3b82f6;
}

.refresh-text {
  font-weight: 500;
}

.refresh-countdown {
  background: #3b82f6;
  color: white;
  padding: 3px 8px;
  border-radius: 12px;
  font-weight: 600;
  min-width: 28px;
  text-align: center;
  font-family: 'SF Mono', 'Monaco', 'Inconsolata', monospace;
  font-size: 0.75rem;
}

/* Timestamp */
.timestamp {
  color: #64748b;
  font-size: 0.75rem;
  display: flex;
  align-items: center;
  gap: 6px;
  width: 100%;
  justify-content: center;
  padding-top: 4px;
}

.timestamp-icon {
  font-size: 0.9em;
}

/* Refresh button */
.btn-refresh {
  width: 50px;
  height: 50px;
  border: none;
  background: #3b82f6; /* Blue background */
  color: white;
  border-radius: 10px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.5em;
  transition: all 0.2s;
  box-shadow: 0 2px 4px rgba(59, 130, 246, 0.3);
}

.btn-refresh:hover:not(:disabled) {
  background: #2563eb;
  transform: rotate(15deg);
  box-shadow: 0 4px 8px rgba(59, 130, 246, 0.4);
}

.btn-refresh:disabled {
  opacity: 0.6;
  cursor: not-allowed;
  transform: none !important;
}

.refresh-icon-spinning {
  animation: spin 1s linear infinite;
}

.timestamp {
  color: #64748b;
  font-size: 0.8rem;
  display: flex;
  align-items: center;
  gap: 6px;
}

.timestamp-icon {
  font-size: 0.9em;
}

/* Main Content */
.main-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  gap: 12px;
}

/* Top Controls */
.top-controls {
  display: flex;
  gap: 12px;
  flex-shrink: 0;
}

.search-container {
  flex: 1;
  position: relative;
}

.search-wrapper {
  position: relative;
  display: flex;
  align-items: center;
}

.search-icon {
  position: absolute;
  left: 12px;
  font-size: 1em;
  color: #94a3b8;
  pointer-events: none;
}

.search-input {
  width: 100%;
  padding: 12px 12px 12px 36px;
  border: 2px solid #e2e8f0;
  border-radius: 10px;
  font-size: 0.9rem;
  transition: all 0.2s;
  background: white;
}

.search-input:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.clear-search {
  position: absolute;
  right: 12px;
  background: none;
  border: none;
  font-size: 1.3em;
  color: #94a3b8;
  cursor: pointer;
  padding: 0;
  width: 20px;
  height: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.clear-search:hover {
  color: #64748b;
}

.action-buttons {
  display: flex;
  gap: 8px;
}

.btn-action {
  padding: 10px 16px;
  border: 2px solid #e2e8f0;
  background: white;
  border-radius: 10px;
  font-size: 0.85rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
  white-space: nowrap;
}

.btn-action:hover {
  background: #f8fafc;
  border-color: #cbd5e1;
}

.btn-action.btn-active {
  background: #3b82f6;
  color: white;
  border-color: #3b82f6;
}

/* Search Results */
.search-results {
  position: absolute;
  top: calc(100% + 4px);
  left: 0;
  right: 0;
  background: white;
  border-radius: 10px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  z-index: 100;
  overflow: hidden;
  border: 1px solid #e2e8f0;
}

.results-scroll {
  max-height: 240px;
  overflow-y: auto;
}

.result-item {
  padding: 12px;
  display: flex;
  align-items: center;
  gap: 12px;
  cursor: pointer;
  transition: background 0.2s;
  border-bottom: 1px solid #f1f5f9;
}

.result-item:hover {
  background: #f8fafc;
}

.result-item:last-child {
  border-bottom: none;
}

.result-flag {
  width: 28px;
  height: 28px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.flag-image {
  width: 100%;
  height: 100%;
  object-fit: cover;
  border-radius: 3px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.special-symbol {
  font-size: 1.5em;
  line-height: 1;
}

.result-details {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.result-details strong {
  font-size: 0.9rem;
  color: #1e293b;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.result-details small {
  font-size: 0.75rem;
  color: #64748b;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.add-btn {
  width: 24px;
  height: 24px;
  background: #3b82f6;
  color: white;
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.9rem;
  font-weight: bold;
  flex-shrink: 0;
}

/* Rates Container */
.rates-container {
  flex: 1;
  background: white;
  border-radius: 12px;
  padding: 16px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.rates-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
  flex-shrink: 0;
}

.rates-title {
  display: flex;
  align-items: center;
  gap: 10px;
}

.rates-title h2 {
  color: #1e293b;
  font-size: 1.1rem;
  font-weight: 600;
}

.rates-count {
  background: #3b82f6;
  color: white;
  width: 24px;
  height: 24px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.8rem;
  font-weight: 600;
}

.view-mode {
  font-size: 0.8rem;
  color: #64748b;
  padding: 4px 8px;
  background: #f1f5f9;
  border-radius: 6px;
}

/* Skeleton Loading */
.skeleton-list {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.skeleton-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px;
  background: #f8fafc;
  border-radius: 10px;
  animation: shimmer 1.5s infinite linear;
  background: linear-gradient(90deg, #f8fafc 25%, #f1f5f9 50%, #f8fafc 75%);
  background-size: 200% 100%;
}

.skeleton-flag {
  width: 32px;
  height: 32px;
  background: #e2e8f0;
  border-radius: 50%;
  flex-shrink: 0;
}

.skeleton-details {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.skeleton-line {
  background: #e2e8f0;
  border-radius: 4px;
}

.skeleton-line-large {
  height: 16px;
  width: 80px;
}

.skeleton-line-small {
  height: 12px;
  width: 60px;
}

.skeleton-rate {
  width: 70px;
  height: 24px;
  background: #e2e8f0;
  border-radius: 6px;
}

/* Empty State */
.empty-state {
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  padding: 40px 20px;
  text-align: center;
}

.empty-icon {
  font-size: 3rem;
  margin-bottom: 16px;
  opacity: 0.5;
}

.empty-state h3 {
  color: #1e293b;
  margin-bottom: 8px;
  font-size: 1.1rem;
}

.empty-state p {
  color: #64748b;
  margin-bottom: 20px;
  font-size: 0.9rem;
}

/* Scrollable rates list */
.rates-list-scrollable {
  flex: 1;
  overflow-y: auto;
  overflow-x: hidden;
  margin-right: -8px;
  padding-right: 8px;
}

.rates-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.rate-card {
  display: flex;
  align-items: center;
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  padding: 12px;
  transition: all 0.2s;
  position: relative;
}

.rate-card:hover {
  border-color: #cbd5e1;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.rate-card-highlight {
  border-color: #3b82f6;
  background: #eff6ff;
}

.rate-card-content {
  flex: 1;
  display: flex;
  justify-content: space-between;
  align-items: center;
  min-height: 44px;
}

.currency-info {
  display: flex;
  align-items: center;
  gap: 12px;
}

.currency-flag {
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.currency-details {
  min-width: 0;
}

.currency-details h3 {
  font-size: 1rem;
  color: #1e293b;
  margin-bottom: 2px;
  font-weight: 600;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.currency-name {
  color: #64748b;
  font-size: 0.8rem;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.rate-display {
  text-align: right;
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  min-width: 120px;
}

.rate-value {
  font-family: 'SF Mono', 'Monaco', 'Inconsolata', monospace;
  font-size: 1rem;
  font-weight: 600;
  color: #1e293b;
  margin-bottom: 2px;
  white-space: nowrap;
}

.rate-meta {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 0.75rem;
}

.rate-change {
  padding: 2px 6px;
  border-radius: 4px;
  font-weight: 500;
  white-space: nowrap;
}

.change-positive {
  background: #dcfce7;
  color: #166534;
}

.change-negative {
  background: #fee2e2;
  color: #991b1b;
}

.change-neutral {
  background: #f1f5f9;
  color: #475569;
}

.rate-trend {
  font-size: 0.9em;
}

.rate-card-actions {
  margin-left: 12px;
  flex-shrink: 0;
}

.action-btn {
  width: 28px;
  height: 28px;
  border: 1px solid #e2e8f0;
  background: white;
  border-radius: 6px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.9rem;
  color: #64748b;
  transition: all 0.2s;
}

.action-btn:hover {
  background: #fee2e2;
  border-color: #fecaca;
  color: #dc2626;
}

/* Toast */
.toast {
  position: fixed;
  bottom: 20px;
  right: 20px;
  background: white;
  border-radius: 10px;
  padding: 12px 16px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  display: flex;
  align-items: center;
  gap: 10px;
  z-index: 1000;
  max-width: 300px;
  animation: slideIn 0.3s ease;
  border-left: 4px solid #3b82f6;
}

.toast-success {
  border-left-color: #22c55e;
}

.toast-error {
  border-left-color: #ef4444;
}

.toast-info {
  border-left-color: #3b82f6;
}

.toast-warning {
  border-left-color: #f59e0b;
}

.toast-icon {
  font-size: 1.1em;
}

.toast-message {
  flex: 1;
  color: #1e293b;
  font-size: 0.85rem;
  font-weight: 500;
}

.toast-close {
  background: none;
  border: none;
  font-size: 1.2em;
  color: #94a3b8;
  cursor: pointer;
  padding: 0;
  width: 18px;
  height: 18px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.toast-close:hover {
  color: #64748b;
}

/* Custom scrollbar */
.rates-list-scrollable::-webkit-scrollbar {
  width: 6px;
}

.rates-list-scrollable::-webkit-scrollbar-track {
  background: #f1f5f9;
  border-radius: 3px;
}

.rates-list-scrollable::-webkit-scrollbar-thumb {
  background: #cbd5e1;
  border-radius: 3px;
}

.rates-list-scrollable::-webkit-scrollbar-thumb:hover {
  background: #94a3b8;
}

/* Animations */
@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

@keyframes shimmer {
  0% { background-position: -200% 0; }
  100% { background-position: 200% 0; }
}

@keyframes slideIn {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

/* Transition Animations */
.list-move {
  transition: transform 0.3s ease;
}

.list-enter-active,
.list-leave-active {
  transition: all 0.3s ease;
}

.list-enter-from,
.list-leave-to {
  opacity: 0;
  transform: translateX(10px);
}

.slide-fade-enter-active,
.slide-fade-leave-active {
  transition: all 0.2s ease;
}

.slide-fade-enter-from,
.slide-fade-leave-to {
  opacity: 0;
  transform: translateY(-5px);
}

.slide-up-enter-active,
.slide-up-leave-active {
  transition: all 0.2s ease;
}

.slide-up-enter-from,
.slide-up-leave-to {
  opacity: 0;
  transform: translateY(10px);
}

/* Responsive Design */
@media (max-width: 768px) {
  .header-top {
    flex-direction: column;
    gap: 16px;
    align-items: stretch;
  }
  
  .header-separator {
    display: none; /* Hide separator on mobile */
  }
  
  .header-controls {
    width: 100%;
    justify-content: space-between;
  }
  
  .status-section {
    min-width: auto;
    width: 100%;
    align-items: center;
  }
  
  h1 {
    font-size: 1.8rem;
    text-align: center;
    justify-content: center;
  }
  
  .currency-logo {
    width: 50px;
    height: 50px;
  }
}

@media (max-width: 480px) {
  .header-controls {
    flex-direction: column;
    gap: 12px;
  }
  
  .btn-refresh {
    align-self: flex-end;
  }
  
  .status-section {
    padding: 10px;
  }
}

@media (min-width: 1400px) {
  .container {
    max-width: 1400px;
    margin: 0 auto;
  }
}
</style>