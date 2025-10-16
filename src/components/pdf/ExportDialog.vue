<template>
  <div v-if="show" class="modal-overlay" @click="$emit('close')">
    <div class="modal-content" @click.stop>
      <h3>Save Signed PDF</h3>
      <div class="dialog-body">
        <label for="export-filename">File Name:</label>
        <input
          id="export-filename"
          type="text"
          v-model="localFilename"
          class="filename-input"
          @keyup.enter="confirm"
        />
      </div>
      <div class="modal-actions">
        <button @click="$emit('close')" class="btn-secondary">Cancel</button>
        <button @click="confirm" class="btn-primary">Download</button>
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
  filename: {
    type: String,
    default: ''
  }
})

const emit = defineEmits(['close', 'confirm'])

const localFilename = ref('')

watch(() => props.show, (newVal) => {
  if (newVal) {
    localFilename.value = props.filename
  }
})

const confirm = () => {
  if (!localFilename.value.trim()) {
    alert('Please enter a filename')
    return
  }
  emit('confirm', localFilename.value)
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

.modal-content h3 {
  margin: 0 0 20px 0;
  font-size: 20px;
}

.dialog-body {
  margin-bottom: 20px;
  min-width: 440px;
}

.dialog-body label {
  display: block;
  margin-bottom: 8px;
  font-weight: 500;
  color: #202124;
}

.filename-input {
  width: 100%;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
  font-family: inherit;
}

.filename-input:focus {
  outline: none;
  border-color: #4285f4;
  box-shadow: 0 0 0 2px rgba(66, 133, 244, 0.1);
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

  .modal-content h3 {
    font-size: 18px;
    margin-bottom: 16px;
  }

  .dialog-body {
    min-width: unset;
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

