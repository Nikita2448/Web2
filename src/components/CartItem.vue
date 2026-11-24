<script setup>
defineProps({
  item: {
    type: Object,
    required: true
  }
})

const emit = defineEmits(['increment', 'decrement', 'remove'])
</script>

<template>
  <div class="cart-item">
    <div class="cart-item-info">
      <h4 class="cart-item-title">{{ item.name }}</h4>
      <span class="cart-item-price">${{ (item.price * item.quantity).toLocaleString() }}</span>
    </div>

    <div class="cart-item-controls">
      <button class="qty-btn" @click="emit('decrement', item.id)">−</button>
      <span class="qty-value">{{ item.quantity }}</span>
      <button class="qty-btn" @click="emit('increment', item.id)">+</button>
      <button class="remove-btn" title="Видалити" @click="emit('remove', item.id)">✕</button>
    </div>
  </div>
</template>