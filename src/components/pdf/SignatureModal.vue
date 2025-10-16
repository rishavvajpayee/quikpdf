<template>
  <div v-if="show" class="modal-overlay" @click="$emit('close')">
    <div class="modal-content" @click.stop>
      <h3>Draw Your Signature</h3>
      <canvas
        ref="signatureCanvas"
        width="800"
        height="300"
        class="signature-canvas"
        @mousedown="startDrawing"
        @mousemove="draw"
        @mouseup="stopDrawing"
        @mouseleave="stopDrawing"
        @touchstart="startDrawing"
        @touchmove="draw"
        @touchend="stopDrawing"
      ></canvas>
      <div class="modal-actions">
        <button @click="clearCanvas" class="btn-secondary">Clear</button>
        <button @click="saveSignature" class="btn-primary">Add to PDF</button>
        <button @click="$emit('close')" class="btn-secondary">Cancel</button>
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
  }
})

const emit = defineEmits(['close', 'save'])

const signatureCanvas = ref(null)
const isDrawing = ref(false)
const signatureContext = ref(null)

watch(() => props.show, (newVal) => {
  if (newVal) {
    setTimeout(() => {
      if (signatureCanvas.value) {
        signatureContext.value = signatureCanvas.value.getContext('2d', { willReadFrequently: true })
        signatureContext.value.strokeStyle = '#000'
        signatureContext.value.lineWidth = 4
        signatureContext.value.lineCap = 'round'
        signatureContext.value.lineJoin = 'round'
        signatureContext.value.imageSmoothingEnabled = true
        signatureContext.value.imageSmoothingQuality = 'high'
      }
    }, 10)
  }
})

const startDrawing = (e) => {
  e.preventDefault()
  isDrawing.value = true
  const rect = signatureCanvas.value.getBoundingClientRect()
  const canvas = signatureCanvas.value
  const scaleX = canvas.width / rect.width
  const scaleY = canvas.height / rect.height
  
  const clientX = e.touches ? e.touches[0].clientX : e.clientX
  const clientY = e.touches ? e.touches[0].clientY : e.clientY
  
  const x = (clientX - rect.left) * scaleX
  const y = (clientY - rect.top) * scaleY

  signatureContext.value.beginPath()
  signatureContext.value.moveTo(x, y)
}

const draw = (e) => {
  if (!isDrawing.value) return
  e.preventDefault()
  const rect = signatureCanvas.value.getBoundingClientRect()
  const canvas = signatureCanvas.value
  const scaleX = canvas.width / rect.width
  const scaleY = canvas.height / rect.height
  
  const clientX = e.touches ? e.touches[0].clientX : e.clientX
  const clientY = e.touches ? e.touches[0].clientY : e.clientY
  
  const x = (clientX - rect.left) * scaleX
  const y = (clientY - rect.top) * scaleY

  signatureContext.value.lineTo(x, y)
  signatureContext.value.stroke()
}

const stopDrawing = () => {
  isDrawing.value = false
}

const clearCanvas = () => {
  if (signatureContext.value && signatureCanvas.value) {
    signatureContext.value.clearRect(0, 0, signatureCanvas.value.width, signatureCanvas.value.height)
  }
}

const saveSignature = () => {
  if (!signatureCanvas.value) return
  const dataUrl = signatureCanvas.value.toDataURL('image/png', 1.0)
  emit('save', dataUrl)
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

.signature-canvas {
  border: 2px solid #ddd;
  border-radius: 4px;
  cursor: crosshair;
  display: block;
  margin: 0 auto 20px auto;
  width: 400px;
  height: 150px;
  image-rendering: high-quality;
  image-rendering: -webkit-optimize-contrast;
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

  .signature-canvas {
    width: 100%;
    max-width: 100%;
    height: 200px;
    margin-bottom: 16px;
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

  .signature-canvas {
    height: 150px;
  }
}
</style>

