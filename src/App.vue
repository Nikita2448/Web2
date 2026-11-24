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

// Стан модального вікна оформлення замовлення
const isCheckoutOpen = ref(false)
const orderSubmitted = ref(false)
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

// Відфільтрований та відсортований список відеокарт
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

    return matchesCategory && matchesBrand && matchesSearch
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

const openCheckout = () => {
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
</script>

<template>
  <AppHeader :cart-count="totalItems" />

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

      <div class="shop-layout">
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
              @add-to-cart="handleAddToCart"
            />
          </div>
        </section>

        <aside class="cart-sidebar">
          <div class="cart-header">
            <h2>Кошик</h2>
            <button v-if="cart.length > 0" class="clear-btn" @click="clearCart">
              Очистити
            </button>
          </div>

          <div v-if="cart.length === 0" class="empty-state">
            <p>Кошик порожній. Додайте відеокарту з каталогу.</p>
          </div>

          <div v-else class="cart-content">
            <div class="cart-list">
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
        </aside>
      </div>
    </div>
  </main>

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
            <input v-model.trim="customer.phone" type="tel" required placeholder="+380 (99) 123-45-67" />
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

  <footer class="site-footer">
    <div class="container">
      <p>© 2026 NV HUB. Навчальний проєкт на Vue 3.</p>
    </div>
  </footer>
</template>