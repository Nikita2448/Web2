<script setup>
defineProps({
  cartCount: {
    type: Number,
    default: 0
  }
})

const emit = defineEmits(['open-cart'])
</script>

<template>
  <header class="site-header">
    <div class="container header-inner">
      <a href="#" class="logo">
        <span class="logo-nv">NV</span> HUB
        <span class="ua-badge">
          <span class="ua-u">U</span><span class="ua-a">A</span>
        </span>
      </a>
      <div class="header-actions">
        <span class="badge-store">GeForce Store</span>
        <button type="button" class="cart-pill cart-trigger" @click="emit('open-cart')">
          Кошик <strong>{{ cartCount }}</strong>
        </button>
      </div>
    </div>
  </header>
</template>