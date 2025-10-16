<template>
  <div v-if="show" class="modal-overlay" @click="$emit('close')">
    <div class="modal-content text-modal" @click.stop>
      <h3>{{ isEditing ? 'Edit Text' : 'Add Text' }}</h3>
      <div class="text-input-container">
        <label for="text-content">Text Content:</label>
        <textarea
          id="text-content"
          v-model="localContent"
          class="text-input"
          placeholder="Enter your text here..."
          rows="4"
          @keydown.enter.ctrl="save"
        ></textarea>
      </div>
      <div class="text-options">
        <div class="option-group">
          <label for="font-size">Font Size:</label>
          <input
            id="font-size"
            type="number"
            v-model.number="localFontSize"
            min="8"
            max="72"
            class="number-input"
          />
        </div>
        <div class="option-group">
          <label for="text-color">Color:</label>
          <input
            id="text-color"
            type="color"
            v-model="localColor"
            class="color-input"
          />
        </div>
        <div class="option-group">
          <label>
            <input type="checkbox" v-model="localBold" />
            Bold
          </label>
        </div>
        <div class="option-group">
          <label>
            <input type="checkbox" v-model="localItalic" />
            Italic
          </label>
        </div>
      </div>
      <div class="modal-actions">
        <button @click="$emit('close')" class="btn-secondary">Cancel</button>
        <button @click="save" class="btn-primary" :disabled="!localContent.trim()">
          {{ isEditing ? 'Update' : 'Add to PDF' }}
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, watch } from 'vue'

const props = defineProps({
  show: {
    type: Boolean,
    default: false
  },
  isEditing: {
    type: Boolean,
    default: false
  },
  content: {
    type: String,
    default: ''
  },
  fontSize: {
    type: Number,
    default: 16
  },
  color: {
    type: String,
    default: '#000000'
  },
  bold: {
    type: Boolean,
    default: false
  },
  italic: {
    type: Boolean,
    default: false
  }
})

const emit = defineEmits(['close', 'save'])

const localContent = ref('')
const localFontSize = ref(16)
const localColor = ref('#000000')
const localBold = ref(false)
const localItalic = ref(false)

// Watch for prop changes to update local state
watch(() => props.show, (newVal) => {
  if (newVal) {
    localContent.value = props.content
    localFontSize.value = props.fontSize
    localColor.value = props.color
    localBold.value = props.bold
    localItalic.value = props.italic
  }
})

const save = () => {
  if (!localContent.value.trim()) return
  
  emit('save', {
    content: localContent.value,
    fontSize: localFontSize.value,
    color: localColor.value,
    bold: localBold.value,
    italic: localItalic.value
  })
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal-content {
  background: white;
  min-width: 500px;
  max-width: 90vw;
  padding: 30px;
  border-radius: 8px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
  overflow: auto;
}

.text-modal {
  min-width: 600px;
}

.modal-content h3 {
  margin: 0 0 20px 0;
  font-size: 20px;
}

.text-input-container {
  margin-bottom: 20px;
}

.text-input-container label {
  display: block;
  margin-bottom: 8px;
  font-weight: 500;
  color: #202124;
}

.text-input {
  width: 100%;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
  font-family: inherit;
  resize: vertical;
}

.text-input:focus {
  outline: none;
  border-color: #4285f4;
  box-shadow: 0 0 0 2px rgba(66, 133, 244, 0.1);
}

.text-options {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 16px;
  padding: 20px;
  background: #f8f9fa;
  border-radius: 4px;
  margin-bottom: 20px;
}

.option-group {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.option-group label {
  font-size: 14px;
  font-weight: 500;
  color: #202124;
  display: flex;
  align-items: center;
  gap: 8px;
}

.number-input {
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
  width: 100%;
}

.number-input:focus {
  outline: none;
  border-color: #4285f4;
  box-shadow: 0 0 0 2px rgba(66, 133, 244, 0.1);
}

.color-input {
  width: 60px;
  height: 38px;
  border: 1px solid #ddd;
  border-radius: 4px;
  cursor: pointer;
}

.color-input:focus {
  outline: none;
  border-color: #4285f4;
  box-shadow: 0 0 0 2px rgba(66, 133, 244, 0.1);
}

.option-group input[type="checkbox"] {
  width: 18px;
  height: 18px;
  cursor: pointer;
}

.modal-actions {
  display: flex;
  gap: 10px;
  justify-content: flex-end;
}

.btn-primary,
.btn-secondary {
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
}

.btn-primary {
  background: #4285f4;
  color: white;
}

.btn-primary:hover {
  background: #3367d6;
}

.btn-primary:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.btn-secondary {
  background: #f1f3f4;
  color: #202124;
}

.btn-secondary:hover {
  background: #e8eaed;
}

@media (max-width: 768px) {
  .modal-content {
    min-width: unset;
    width: calc(100vw - 24px);
    max-width: calc(100vw - 24px);
    padding: 20px;
    margin: 12px;
    max-height: 90vh;
  }

  .text-modal {
    min-width: unset;
  }

  .modal-content h3 {
    font-size: 18px;
    margin-bottom: 16px;
  }

  .text-options {
    grid-template-columns: 1fr;
    gap: 12px;
    padding: 16px;
  }

  .modal-actions {
    flex-direction: column-reverse;
    gap: 8px;
  }

  .modal-actions button {
    width: 100%;
  }
}

@media (max-width: 480px) {
  .modal-content {
    padding: 16px;
    margin: 8px;
  }
}
</style>

