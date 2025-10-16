<template>
  <div v-if="show" class="pdf-not-loaded">
    <div class="empty-state">
      <h3>No PDF Document Loaded</h3>
      <p>Upload a PDF document to start editing.</p>
      <button @click="$emit('load-pdf')" class="load-pdf-btn">
        Load PDF Document
      </button>
    </div>
  </div>
</template>

<script setup>
defineProps({
  show: {
    type: Boolean,
    default: false
  }
})

defineEmits(['load-pdf'])
</script>

<style scoped>
.pdf-not-loaded {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
  padding: 40px;
}

.empty-state {
  text-align: center;
  max-width: 400px;
}

.empty-state h3 {
  font-size: 24px;
  font-weight: 500;
  color: #202124;
  margin: 0 0 12px 0;
}

.empty-state p {
  font-size: 16px;
  color: #5f6368;
  margin: 0 0 32px 0;
  line-height: 1.5;
}

.load-pdf-btn {
  display: inline-flex;
  align-items: center;
  gap: 10px;
  padding: 14px 28px;
  background: #4285f4;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 16px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
  box-shadow: 0 2px 4px rgba(66, 133, 244, 0.3);
}

.load-pdf-btn:hover {
  background: #3367d6;
  box-shadow: 0 4px 8px rgba(66, 133, 244, 0.4);
  transform: translateY(-1px);
}

.load-pdf-btn:active {
  transform: translateY(0);
  box-shadow: 0 2px 4px rgba(66, 133, 244, 0.3);
}

@media (max-width: 768px) {
  .empty-state h3 {
    font-size: 20px;
  }

  .empty-state p {
    font-size: 14px;
  }

  .load-pdf-btn {
    padding: 12px 24px;
    font-size: 14px;
    width: 100%;
  }
}
</style>

