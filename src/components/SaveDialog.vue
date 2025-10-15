<template>
  <div v-if="show" class="dialog-overlay" @click="$emit('close')">
    <div class="dialog-box" @click.stop>
      <div class="dialog-header">
        <h3>Save Document</h3>
        <button class="close-btn" @click="$emit('close')">×</button>
      </div>
      <div class="dialog-content">
        <div class="input-group">
          <label for="filename">File Name:</label>
          <input 
            type="text" 
            id="filename" 
            :value="filename" 
            @input="$emit('update:filename', $event.target.value)"
            placeholder="Enter file name"
            @keyup.enter="$emit('download')"
          />
        </div>
        <p class="dialog-info">Your document will be saved as a PDF file.</p>
      </div>
      <div class="dialog-footer">
        <button class="btn-cancel" @click="$emit('close')">Cancel</button>
        <button class="btn-download" @click="$emit('download')">
          <span>📥</span> Download PDF
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
defineProps({
  show: Boolean,
  filename: String
})

defineEmits(['close', 'download', 'update:filename'])
</script>

<style scoped>
.dialog-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.dialog-box {
  background: white;
  border-radius: 8px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
  width: 90%;
  max-width: 500px;
  overflow: hidden;
}

.dialog-header {
  padding: 20px 24px;
  border-bottom: 1px solid #e0e0e0;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.dialog-header h3 {
  margin: 0;
  font-size: 20px;
  font-weight: 500;
  color: #202124;
}

.close-btn {
  background: none;
  border: none;
  font-size: 28px;
  color: #5f6368;
  cursor: pointer;
  padding: 0;
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 4px;
}

.close-btn:hover {
  background-color: #f1f3f4;
}

.dialog-content {
  padding: 24px;
}

.input-group {
  margin-bottom: 16px;
}

.input-group label {
  display: block;
  margin-bottom: 8px;
  font-size: 14px;
  font-weight: 500;
  color: #202124;
}

.input-group input {
  width: 100%;
  padding: 10px 12px;
  border: 1px solid #e0e0e0;
  border-radius: 4px;
  font-size: 14px;
  outline: none;
  transition: border-color 0.2s;
}

.input-group input:focus {
  border-color: #1a73e8;
}

.dialog-info {
  margin: 0;
  font-size: 14px;
  color: #5f6368;
}

.dialog-footer {
  padding: 16px 24px;
  border-top: 1px solid #e0e0e0;
  display: flex;
  justify-content: flex-end;
  gap: 12px;
}

.btn-cancel,
.btn-download {
  padding: 10px 24px;
  border: none;
  border-radius: 4px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: background-color 0.2s;
}

.btn-cancel {
  background-color: transparent;
  color: #1a73e8;
}

.btn-cancel:hover {
  background-color: #f1f3f4;
}

.btn-download {
  background-color: #1a73e8;
  color: white;
  display: flex;
  align-items: center;
  gap: 8px;
}

.btn-download:hover {
  background-color: #1765cc;
}

.btn-download span {
  font-size: 16px;
}
</style>
