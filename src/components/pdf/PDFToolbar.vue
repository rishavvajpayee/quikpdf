<template>
  <div class="pdf-toolbar">
    <input
      type="file"
      ref="fileInput"
      @change="handleFileChange"
      accept=".pdf"
      style="display: none"
    />
    <button @click="$emit('add-signature')" class="toolbar-btn" :disabled="!pdfLoaded">
      Add Signature
    </button>
    <button @click="$emit('add-text')" class="toolbar-btn" :disabled="!pdfLoaded">
      Add Text
    </button>
    <button @click="$emit('clear-all')" class="toolbar-btn" :disabled="!hasAnnotations">
      Clear All
    </button>
  </div>
</template>

<script setup>
import { ref } from 'vue'

defineProps({
  pdfLoaded: {
    type: Boolean,
    default: false
  },
  hasAnnotations: {
    type: Boolean,
    default: false
  }
})

const emit = defineEmits(['add-signature', 'add-text', 'clear-all', 'file-selected'])

const fileInput = ref(null)

const handleFileChange = (event) => {
  emit('file-selected', event)
}

// Expose method to parent
defineExpose({
  clickFileInput: () => fileInput.value?.click()
})
</script>

<style scoped>
.pdf-toolbar {
  display: flex;
  gap: 10px;
  padding: 10px 20px;
  background: white;
  border-bottom: 1px solid #e0e0e0;
}

.toolbar-btn {
  padding: 8px 16px;
  background: #4285f4;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
}

.toolbar-btn:hover:not(:disabled) {
  background: #3367d6;
}

.toolbar-btn:disabled {
  background: #ccc;
  cursor: not-allowed;
}

@media (max-width: 768px) {
  .pdf-toolbar {
    padding: 8px 12px;
    gap: 6px;
    flex-wrap: wrap;
  }

  .toolbar-btn {
    padding: 8px 12px;
    font-size: 13px;
    flex: 1;
    min-width: calc(50% - 3px);
  }
}

@media (max-width: 480px) {
  .pdf-toolbar {
    padding: 6px 10px;
    gap: 4px;
  }

  .toolbar-btn {
    padding: 6px 10px;
    font-size: 12px;
    min-width: calc(50% - 2px);
  }
}
</style>

