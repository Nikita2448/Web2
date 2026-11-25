<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import AppHeader from './components/AppHeader.vue'
import ProductCard from './components/ProductCard.vue'
import CartItem from './components/CartItem.vue'

const CART_STORAGE_KEY = 'nvhub_cart_v1'

const products = ref([])
const cart = ref([])
const isLoading = ref(true)
const error = ref(null)

// Стан пошуку, фільтрів та сортування
const searchQuery = ref('')
const selectedCategory = ref('All')
const selectedBrand = ref('All')
const sortBy = ref('default')

const categories = ['All', 'RTX 50', 'RTX 40', 'RTX 30']
const brands = ['All', 'NVIDIA', 'ASUS', 'MSI', 'Gigabyte', 'Palit']

// Стан підбору відеокарти під конфігурацію ПК
const isConfiguratorOpen = ref(false)
const pcBuild = ref({
  active: false,
  cpuTier: 4,
  psuWattage: 1000,
  maxLengthMm: 380,
  targetRes: 'all' // 'all' | '1080p' | '1440p' | '4k'
})

// Стан модальних вікон (кошик та оформлення замовлення)
const isCartOpen = ref(false)
const isCheckoutOpen = ref(false)
const orderSubmitted = ref(false)
const selectedGpu = ref(null)
const customer = ref({
  name: '',
  phone: '',
  city: '',
  delivery: 'Нова Пошта'
})

onMounted(async () => {
  const savedCart = localStorage.getItem(CART_STORAGE_KEY)
  if (savedCart) {
    try {
      cart.value = JSON.parse(savedCart)
    } catch {
      localStorage.removeItem(CART_STORAGE_KEY)
    }
  }

  try {
    const response = await fetch('/data/gpus.json')
    if (!response.ok) throw new Error(`HTTP ${response.status}`)
    products.value = await response.json()
  } catch (err) {
    error.value = 'Не вдалося завантажити каталог відеокарт.'
    console.error(err)
  } finally {
    isLoading.value = false
  }
})

watch(
  cart,
  (newCart) => {
    localStorage.setItem(CART_STORAGE_KEY, JSON.stringify(newCart))
  },
  { deep: true }
)

watch(
  [isCartOpen, isCheckoutOpen, isConfiguratorOpen, selectedGpu],
  ([cartOpen, checkoutOpen, configuratorOpen, gpu]) => {
    const isAnyModalOpen = Boolean(cartOpen || checkoutOpen || configuratorOpen || gpu)
    document.body.style.overflow = isAnyModalOpen ? 'hidden' : ''
  }
)

const filteredProducts = computed(() => {
  const query = searchQuery.value.trim().toLowerCase()

  const result = products.value.filter((gpu) => {
    const matchesCategory =
      selectedCategory.value === 'All' || gpu.category === selectedCategory.value
    const matchesBrand =
      selectedBrand.value === 'All' || gpu.brand === selectedBrand.value
    const matchesSearch =
      !query ||
      gpu.name.toLowerCase().includes(query) ||
      gpu.brand.toLowerCase().includes(query) ||
      gpu.architecture.toLowerCase().includes(query)

    if (!matchesCategory || !matchesBrand || !matchesSearch) return false

    // Фільтрація за параметрами ПК користувача
    if (pcBuild.value.active) {
      const reqCpuTier = getRequiredCpuTier(gpu)
      const reqPsu = parseSpecNumber(gpu.recommendedPsu, 550)
      const lengthMm = parseSpecNumber(gpu.dimensions, 300)
      const vramGb = parseSpecNumber(gpu.vram, 8)

      if (reqCpuTier > pcBuild.value.cpuTier) return false
      if (reqPsu > pcBuild.value.psuWattage) return false
      if (lengthMm > pcBuild.value.maxLengthMm) return false

      if (pcBuild.value.targetRes === '1440p' && vramGb < 12) return false
      if (pcBuild.value.targetRes === '4k' && vramGb < 16) return false
    }

    return true
  })

  if (sortBy.value === 'price-asc') {
    return [...result].sort((a, b) => a.price - b.price)
  }
  if (sortBy.value === 'price-desc') {
    return [...result].sort((a, b) => b.price - a.price)
  }

  return result
})

const resetFilters = () => {
  searchQuery.value = ''
  selectedCategory.value = 'All'
  selectedBrand.value = 'All'
  sortBy.value = 'default'
  resetPcBuild()
}

// Витягує перше число з рядка (напр. "750 Вт" -> 750, "312 мм (2.5 слоти)" -> 312)
const parseSpecNumber = (str, fallback) => {
  if (!str) return fallback
  const match = String(str).match(/\d+/)
  return match ? Number(match[0]) : fallback
}

// Визначає мінімально рекомендований рівень процесора (1..4) для відеокарти
const getRequiredCpuTier = (gpu) => {
  const name = gpu.name.toUpperCase()
  if (name.includes('5090') || name.includes('4090')) return 4
  if (name.includes('5080') || name.includes('4080') || name.includes('4070 TI')) return 3
  if (name.includes('5070') || name.includes('4070') || name.includes('3080') || name.includes('3090')) return 2
  return 1
}

const applyPcBuild = () => {
  pcBuild.value.active = true
  isConfiguratorOpen.value = false
}

const resetPcBuild = () => {
  pcBuild.value = {
    active: false,
    cpuTier: 4,
    psuWattage: 1000,
    maxLengthMm: 380,
    targetRes: 'all'
  }
}

const totalItems = computed(() => {
  return cart.value.reduce((sum, item) => sum + item.quantity, 0)
})

const totalPrice = computed(() => {
  return cart.value.reduce((sum, item) => sum + item.price * item.quantity, 0)
})

const handleAddToCart = (gpu) => {
  const existingItem = cart.value.find((item) => item.id === gpu.id)
  if (existingItem) {
    existingItem.quantity += 1
  } else {
    cart.value.push({ ...gpu, quantity: 1 })
  }
}

const getItemQty = (gpuId) => {
  const found = cart.value.find((item) => item.id === gpuId)
  return found ? found.quantity : 0
}

const incrementQuantity = (id) => {
  const item = cart.value.find((item) => item.id === id)
  if (item) item.quantity += 1
}

const decrementQuantity = (id) => {
  const item = cart.value.find((item) => item.id === id)
  if (!item) return

  if (item.quantity > 1) {
    item.quantity -= 1
  } else {
    removeFromCart(id)
  }
}

const removeFromCart = (id) => {
  cart.value = cart.value.filter((item) => item.id !== id)
}

const clearCart = () => {
  cart.value = []
}

const openCart = () => {
  isCartOpen.value = true
}

const closeCart = () => {
  isCartOpen.value = false
}

const openCheckout = () => {
  isCartOpen.value = false
  orderSubmitted.value = false
  isCheckoutOpen.value = true
}

const closeCheckout = () => {
  isCheckoutOpen.value = false
}

const submitOrder = () => {
  orderSubmitted.value = true
  clearCart()
}

const sanitizePhone = (event) => {
  let cleaned = event.target.value.replace(/[^\d+\s()-]/g, '')
  // Видаляємо всі символи '+', окрім найпершого на початку рядка
  cleaned = cleaned.replace(/(?!^)\+/g, '')
  customer.value.phone = cleaned
  event.target.value = cleaned
}

const openProductDetails = (gpu) => {
  selectedGpu.value = gpu
}

const closeProductDetails = () => {
  selectedGpu.value = null
}
</script>

<template>
  <AppHeader :cart-count="totalItems" @open-cart="openCart" />

  <main class="container">
    <section class="hero">
      <h1>Каталог відеокарт NVIDIA GeForce</h1>
      <p>Лінійка від RTX 3060 до флагманських RTX 5090 від провідних вендорів.</p>
    </section>

    <div v-if="isLoading" class="empty-state">
      <p>Завантаження каталогу...</p>
    </div>

    <div v-else-if="error" class="empty-state">
      <p>{{ error }}</p>
    </div>

    <div v-else>
      <!-- Панель пошуку, фільтрів та сортування -->
      <section class="filters-bar">
        <div class="filters-top">
          <input
            v-model="searchQuery"
            type="search"
            class="search-input"
            placeholder="Пошук моделі (напр. 5080, ROG, SUPRIM, Blackwell)..."
          />

          <select v-model="sortBy" class="sort-select">
            <option value="default">Сортування: За замовчуванням</option>
            <option value="price-asc">Ціна: Від дешевших до дорожчих</option>
            <option value="price-desc">Ціна: Від дорожчих до дешевших</option>
          </select>

          <button
            type="button"
            :class="['btn-configurator', { active: pcBuild.active }]"
            @click="isConfiguratorOpen = true"
          >
            {{ pcBuild.active ? `✓ Підбір: CPU Рівень ${pcBuild.cpuTier} / БЖ ${pcBuild.psuWattage}Вт` : 'Підбір під мій ПК' }}
          </button>
        </div>

        <div class="filters-groups">
          <div class="filter-chips">
            <span class="filter-label">Серія:</span>
            <button
              v-for="cat in categories"
              :key="cat"
              :class="['chip-btn', { active: selectedCategory === cat }]"
              @click="selectedCategory = cat"
            >
              {{ cat === 'All' ? 'Усі' : cat }}
            </button>
          </div>

          <div class="filter-chips">
            <span class="filter-label">Бренд:</span>
            <button
              v-for="brand in brands"
              :key="brand"
              :class="['chip-btn', { active: selectedBrand === brand }]"
              @click="selectedBrand = brand"
            >
              {{ brand === 'All' ? 'Усі' : brand }}
            </button>
          </div>
        </div>
      </section>

      <!-- Каталог на всю ширину -->
      <section>
        <div v-if="filteredProducts.length === 0" class="empty-state">
          <p>За вибраними параметрами відеокарт не знайдено.</p>
          <button class="btn-primary" style="margin-top: 12px" @click="resetFilters">
            Скинути фільтри
          </button>
        </div>

        <div v-else class="products-grid">
          <ProductCard
            v-for="gpu in filteredProducts"
            :key="gpu.id"
            :gpu="gpu"
            :in-cart-qty="getItemQty(gpu.id)"
            @add-to-cart="handleAddToCart"
            @select-gpu="openProductDetails"
          />
        </div>
      </section>
    </div>
  </main>

  <!-- Модальне вікно кошика -->
  <div v-if="isCartOpen" class="modal-backdrop" @click.self="closeCart">
    <div class="modal-card modal-cart">
      <button class="modal-close" @click="closeCart">✕</button>

      <div class="cart-modal-header">
        <h2>Кошик</h2>
        <button v-if="cart.length > 0" class="clear-btn" @click="clearCart">
          Очистити
        </button>
      </div>

      <div v-if="cart.length === 0" class="empty-state">
        <p>Кошик порожній. Додайте відеокарту з каталогу.</p>
        <button class="btn-primary" style="margin-top: 16px" @click="closeCart">
          Перейти до вибору
        </button>
      </div>

      <div v-else class="cart-content">
        <div class="cart-list modal-cart-list">
          <CartItem
            v-for="item in cart"
            :key="item.id"
            :item="item"
            @increment="incrementQuantity"
            @decrement="decrementQuantity"
            @remove="removeFromCart"
          />
        </div>

        <div class="cart-summary">
          <div class="summary-row">
            <span>Разом ({{ totalItems }} шт.):</span>
            <strong>${{ totalPrice.toLocaleString() }}</strong>
          </div>
          <button class="btn-primary checkout-btn" @click="openCheckout">
            Оформити замовлення
          </button>
        </div>
      </div>
    </div>
  </div>

  <!-- Модальне вікно оформлення замовлення -->
  <div v-if="isCheckoutOpen" class="modal-backdrop" @click.self="closeCheckout">
    <div class="modal-card">
      <button class="modal-close" @click="closeCheckout">✕</button>

      <div v-if="!orderSubmitted">
        <h2>Оформлення замовлення</h2>
        <p class="modal-subtitle">
          Товарів: <strong>{{ totalItems }} шт.</strong> на суму
          <strong>${{ totalPrice.toLocaleString() }}</strong>
        </p>

        <form class="checkout-form" @submit.prevent="submitOrder">
          <label>
            Ім'я та прізвище
            <input v-model.trim="customer.name" type="text" required placeholder="Олександр Коваленко" />
          </label>

          <label>
            Телефон
            <input
              :value="customer.phone"
              type="tel"
              inputmode="tel"
              required
              pattern="[\d+\s()-]{10,20}"
              title="Введіть коректний номер телефону (мінімум 10 символів, без літер)"
              placeholder="+380 (99) 123-45-67"
              @input="sanitizePhone"
            />
          </label>

          <label>
            Місто доставки
            <input v-model.trim="customer.city" type="text" required placeholder="Київ" />
          </label>

          <label>
            Спосіб доставки
            <select v-model="customer.delivery">
              <option value="Нова Пошта">Нова Пошта (відділення / поштомат)</option>
              <option value="Кур'єр">Кур'єрська доставка</option>
              <option value="Самовивіз">Самовивіз із шоуруму NV HUB</option>
            </select>
          </label>

          <button type="submit" class="btn-primary checkout-btn">
            Підтвердити замовлення
          </button>
        </form>
      </div>

      <div v-else class="order-success">
        <div class="success-icon">✓</div>
        <h2>Замовлення прийнято!</h2>
        <p>Дякуємо, <strong>{{ customer.name }}</strong>. Ми зв'яжемося за номером {{ customer.phone }} для підтвердження доставки ({{ customer.city }}, {{ customer.delivery }}).</p>
        <button class="btn-primary" @click="closeCheckout">Повернутися до каталогу</button>
      </div>
    </div>
  </div>

<!-- Модальне вікно детального опису відеокарти -->
  <div v-if="selectedGpu" class="modal-backdrop" @click.self="closeProductDetails">
    <div class="modal-card modal-details">
      <button class="modal-close" @click="closeProductDetails">✕</button>

      <div class="details-image">
        <img :src="selectedGpu.image" :alt="selectedGpu.name" />
        <span v-if="selectedGpu.badge" class="card-badge">{{ selectedGpu.badge }}</span>
      </div>

      <div class="details-body">
        <span class="card-category">
          {{ selectedGpu.brand }} • {{ selectedGpu.category }} • Архітектура {{ selectedGpu.architecture }}
        </span>
        <h2>{{ selectedGpu.name }}</h2>

        <p class="details-description">
          {{ selectedGpu.description || `Фірмова відеокарта ${selectedGpu.name} на базі архітектури NVIDIA ${selectedGpu.architecture}. Оснащена ${selectedGpu.vram} високошвидкісної відеопам'яті та ${selectedGpu.cudaCores.toLocaleString()} ядрами CUDA для максимальної продуктивності в іграх із трасуванням променів та роботі з ШІ.` }}
        </p>

        <div class="specs-grid">
          <div class="spec-box">
            <span class="spec-label">Відеопам'ять</span>
            <strong>{{ selectedGpu.vram }}</strong>
          </div>
          <div class="spec-box">
            <span class="spec-label">Ядра CUDA</span>
            <strong>{{ selectedGpu.cudaCores.toLocaleString() }}</strong>
          </div>
          <div class="spec-box">
            <span class="spec-label">Споживання (TDP)</span>
            <strong>{{ selectedGpu.tdp || '285 Вт' }}</strong>
          </div>
          <div class="spec-box">
            <span class="spec-label">Рекомендований БЖ</span>
            <strong>{{ selectedGpu.recommendedPsu || '750 Вт' }}</strong>
          </div>
          <div class="spec-box">
            <span class="spec-label">Рекомендований CPU</span>
            <strong>{{ selectedGpu.recommendedCpu || 'Ryzen 7 7800X3D / Core i7-14700K' }}</strong>
          </div>
          <div class="spec-box">
            <span class="spec-label">Роз'єм живлення</span>
            <strong>{{ selectedGpu.powerConnector || '1× 16-pin (12V-2x6)' }}</strong>
          </div>
          <div class="spec-box">
            <span class="spec-label">Довжина / товщина</span>
            <strong>{{ selectedGpu.dimensions || '312 мм (2.5 слоти)' }}</strong>
          </div>
          <div class="spec-box">
            <span class="spec-label">Інтерфейс</span>
            <strong>{{ selectedGpu.interface || 'PCIe 5.0 x16' }}</strong>
          </div>
        </div>

        <div class="details-footer">
          <div class="details-price">
            <span>Ціна:</span>
            <strong>${{ selectedGpu.price.toLocaleString() }}</strong>
          </div>

          <button
            :class="['btn-primary', { 'btn-in-cart': getItemQty(selectedGpu.id) > 0 }]"
            @click="handleAddToCart(selectedGpu)"
          >
            {{ getItemQty(selectedGpu.id) > 0 ? `✓ У кошику (${getItemQty(selectedGpu.id)}) — додати ще` : 'Додати в кошик' }}
          </button>
        </div>
      </div>
    </div>
  </div>

  <div v-if="isConfiguratorOpen" class="modal-backdrop" @click.self="isConfiguratorOpen = false">
    <div class="modal-card">
      <button class="modal-close" @click="isConfiguratorOpen = false">✕</button>
      <h2>Підбір відеокарти під конфігурацію</h2>
      <p class="modal-subtitle">Вкажіть параметри вашого ПК — ми залишимо лише сумісні моделі.</p>

      <form class="checkout-form" @submit.prevent="applyPcBuild">
        <label>
          Ваш процесор (CPU)
          <select v-model.number="pcBuild.cpuTier">
            <option :value="1">Початковий (Ryzen 5 3600/5500, Core i3-12100F / i5-10400F)</option>
            <option :value="2">Середній (Ryzen 5 5600/7500F, Core i5-12400F / 13400F)</option>
            <option :value="3">Продуктивний (Ryzen 7 5800X3D/7700X, Core i5-13600K / i7-13700K)</option>
            <option :value="4">Флагманський (Ryzen 7 7800X3D/9800X3D, Core i7-14700K / i9 / Ultra 9)</option>
          </select>
        </label>

        <label>
          Потужність вашого блоку живлення (БЖ)
          <select v-model.number="pcBuild.psuWattage">
            <option :value="550">550 Вт</option>
            <option :value="650">650 Вт</option>
            <option :value="750">750 Вт</option>
            <option :value="850">850 Вт</option>
            <option :value="1000">1000+ Вт</option>
          </select>
        </label>

        <label>
          Макс. довжина відеокарти для корпусу: <strong>{{ pcBuild.maxLengthMm }} мм</strong>
          <input v-model.number="pcBuild.maxLengthMm" type="range" min="240" max="380" step="5" />
        </label>

        <label>
          Цільова роздільна здатність / задачі
          <select v-model="pcBuild.targetRes">
            <option value="all">Будь-яка (усі моделі)</option>
            <option value="1080p">Full HD (1080p) — від 8 ГБ VRAM</option>
            <option value="1440p">Quad HD (1440p) — від 12 ГБ VRAM</option>
            <option value="4k">4K Ultra / Робота з ШІ — від 16 ГБ VRAM</option>
          </select>
        </label>

        <div class="configurator-actions">
          <button type="submit" class="btn-primary">
            Показати сумісні карти
          </button>
          <button
            v-if="pcBuild.active"
            type="button"
            class="clear-btn"
            @click="resetPcBuild(); isConfiguratorOpen = false"
          >
            Скинути
          </button>
        </div>
      </form>
    </div>
  </div>

  <footer class="site-footer">
    <div class="container">
      <p>© 2026 NV HUB. Навчальний проєкт на Vue 3.</p>
    </div>
  </footer>
</template>