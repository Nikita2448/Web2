<script setup>
const props = defineProps({
  gpu: {
    type: Object,
    required: true
  },
  inCartQty: {
    type: Number,
    default: 0
  }
})

const emit = defineEmits(['add-to-cart', 'select-gpu'])

const handleImageError = (event) => {
  event.target.src = `https://placehold.co/400x260/141414/76b900?text=${encodeURIComponent(props.gpu.brand + ' ' + props.gpu.category)}`
}
</script>

<template>
  <article class="product-card" @click="emit('select-gpu', gpu)">
    <span v-if="gpu.badge" class="card-badge">{{ gpu.badge }}</span>

    <div class="card-image">
      <img :src="gpu.image" :alt="gpu.name" @error="handleImageError" />
    </div>

    <div class="card-body">
      <span class="card-category">{{ gpu.brand }} • {{ gpu.category }} • {{ gpu.architecture }}</span>
      <h2 class="card-title">{{ gpu.name }}</h2>
      <p class="card-specs">Пам'ять: {{ gpu.vram }} | CUDA: {{ gpu.cudaCores.toLocaleString() }}</p>

      <div class="card-footer">
        <span class="card-price">${{ gpu.price.toLocaleString() }}</span>
        <button
          :class="['btn-primary', { 'btn-in-cart': inCartQty > 0 }]"
          @click.stop="emit('add-to-cart', gpu)"
        >
          {{ inCartQty > 0 ? `✓ У кошику (${inCartQty})` : 'У кошик' }}
        </button>
      </div>
    </div>
  </article>
</template>