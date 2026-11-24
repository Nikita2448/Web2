<script setup>
defineProps({
  cartCount: {
    type: Number,
    default: 0
  }
})
</script>

<template>
  <header class="site-header">
    <div class="container header-inner">
      <a href="#" class="logo">NV <span>HUB</span></a>
      <div class="header-actions">
        <span class="badge">GeForce Store</span>
        <span class="cart-pill">Кошик: <strong>{{ cartCount }}</strong></span>
      </div>
    </div>
  </header>
</template>