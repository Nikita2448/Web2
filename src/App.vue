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
  // Відновлюємо кошик з localStorage
  const savedCart = localStorage.getItem(CART_STORAGE_KEY)
  if (savedCart) {
    try {
      cart.value = JSON.parse(savedCart)
    } catch {
      localStorage.removeItem(CART_STORAGE_KEY)
    }
  }

  // Завантажуємо каталог
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

// Автоматично зберігаємо кошик при будь-якій зміні
watch(
  cart,
  (newCart) => {
    localStorage.setItem(CART_STORAGE_KEY, JSON.stringify(newCart))
  },
  { deep: true }
)

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

    <div v-else class="shop-layout">
      <section class="products-grid">
        <ProductCard
          v-for="gpu in products"
          :key="gpu.id"
          :gpu="gpu"
          @add-to-cart="handleAddToCart"
        />
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