<template>
  <div v-if="show" class="placement-hint">
    📍 Click anywhere on the PDF to place your {{ type }}
  </div>
</template>

<script setup>
defineProps({
  show: {
    type: Boolean,
    default: false
  },
  type: {
    type: String,
    default: 'element'
  }
})
</script>

<style scoped>
.placement-hint {
  position: fixed;
  top: 100px;
  left: 50%;
  transform: translateX(-50%);
  background: #4285f4;
  color: white;
  padding: 12px 24px;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
  z-index: 999;
  font-size: 14px;
  font-weight: 500;
  animation: slideDown 0.3s ease-out;
}

@keyframes slideDown {
  from {
    opacity: 0;
    transform: translateX(-50%) translateY(-20px);
  }
  to {
    opacity: 1;
    transform: translateX(-50%) translateY(0);
  }
}

@media (max-width: 768px) {
  .placement-hint {
    top: 80px;
    left: 12px;
    right: 12px;
    transform: none;
    padding: 10px 16px;
    font-size: 13px;
    text-align: center;
  }

  @keyframes slideDown {
    from {
      opacity: 0;
      transform: translateY(-20px);
    }
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }
}

@media (max-width: 480px) {
  .placement-hint {
    font-size: 12px;
    padding: 8px 12px;
  }
}
</style>

