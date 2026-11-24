<script setup>
import { ref, computed, onMounted } from 'vue'
import AppHeader from './components/AppHeader.vue'
import ProductCard from './components/ProductCard.vue'
import CartItem from './components/CartItem.vue'

const products = ref([])
const cart = ref([])
const isLoading = ref(true)
const error = ref(null)

onMounted(async () => {
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

// Загальна кількість одиниць товару в кошику (для шапки)
const totalItems = computed(() => {
  return cart.value.reduce((sum, item) => sum + item.quantity, 0)
})

// Загальна вартість усіх товарів у кошику
const totalPrice = computed(() => {
  return cart.value.reduce((sum, item) => sum + item.price * item.quantity, 0)
})

// Додавання відеокарти в кошик
const handleAddToCart = (gpu) => {
  const existingItem = cart.value.find((item) => item.id === gpu.id)
  if (existingItem) {
    existingItem.quantity += 1
  } else {
    cart.value.push({ ...gpu, quantity: 1 })
  }
}

// Збільшити кількість (+)
const incrementQuantity = (id) => {
  const item = cart.value.find((item) => item.id === id)
  if (item) item.quantity += 1
}

// Зменшити кількість (−)
const decrementQuantity = (id) => {
  const item = cart.value.find((item) => item.id === id)
  if (!item) return

  if (item.quantity > 1) {
    item.quantity -= 1
  } else {
    removeFromCart(id)
  }
}

// Видалити позицію з кошика (✕)
const removeFromCart = (id) => {
  cart.value = cart.value.filter((item) => item.id !== id)
}

// Очистити весь кошик
const clearCart = () => {
  cart.value = []
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
          </div>
        </div>
      </aside>
    </div>
  </main>

  <footer class="site-footer">
    <div class="container">
      <p>© 2026 NV HUB. Навчальний проєкт на Vue 3.</p>
    </div>
  </footer>
</template>