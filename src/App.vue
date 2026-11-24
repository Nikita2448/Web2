<script setup>
import { ref, onMounted } from 'vue'
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

const handleAddToCart = (gpu) => {
  console.log('Обрано:', gpu.name)
}
</script>

<template>
  <AppHeader :cart-count="cart.length" />

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
        <h2>Кошик</h2>
        <div v-if="cart.length === 0" class="empty-state">
          <p>Кошик порожній. Додайте відеокарту з каталогу.</p>
        </div>
        <div v-else class="cart-list">
          <CartItem
            v-for="item in cart"
            :key="item.id"
            :item="item"
          />
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