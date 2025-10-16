<template>
  <div class="pdf-canvas-wrapper">
    <div class="pdf-scroll-container">
      <div 
        class="pdf-container" 
        :style="{ transform: `scale(${zoom})` }" 
        @click="handleContainerClick"
      >
        <!-- Multiple page canvases -->
        <div
          v-for="pageNum in totalPages"
          :key="pageNum"
          class="pdf-page-wrapper"
          @click="$emit('place-element', $event)"
          @touchend="$emit('place-element', $event)"
        >
          <canvas
            :ref="el => setPageRef(el, pageNum)"
            class="pdf-canvas"
          ></canvas>
        </div>

        <!-- Signature overlays -->
        <div
          v-for="(sig, index) in signatures"
          :key="index"
          class="signature-overlay"
          :class="{ 'selected': selectedType === 'signature' && selectedIndex === index }"
          :style="{
            left: sig.x + 'px',
            top: sig.y + 'px',
            width: sig.width + 'px',
            height: sig.height + 'px'
          }"
          @mousedown="$emit('drag-start', $event, index, 'signature')"
          @touchstart="$emit('drag-start', $event, index, 'signature')"
          @click="$emit('select-element', index, 'signature', $event)"
        >
          <img :src="sig.image" alt="Signature" style="width: 100%; height: 100%;" />
          <button class="delete-sig" @click="$emit('delete-signature', index)">×</button>
        </div>

        <!-- Text overlays -->
        <div
          v-for="(text, index) in textAnnotations"
          :key="'text-' + index"
          class="text-overlay"
          :class="{ 'selected': selectedType === 'text' && selectedIndex === index }"
          :style="{
            left: text.x + 'px',
            top: text.y + 'px',
            fontSize: text.fontSize + 'px',
            color: text.color,
            fontWeight: text.bold ? 'bold' : 'normal',
            fontStyle: text.italic ? 'italic' : 'normal'
          }"
          @mousedown="$emit('drag-start', $event, index, 'text')"
          @touchstart="$emit('drag-start', $event, index, 'text')"
          @click="$emit('select-element', index, 'text', $event)"
          @dblclick="$emit('edit-text', index)"
        >
          <span>{{ text.content }}</span>
          <button class="delete-sig" @click="$emit('delete-text', index)">×</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, watch, onMounted } from 'vue'

const props = defineProps({
  zoom: {
    type: Number,
    default: 1
  },
  totalPages: {
    type: Number,
    default: 0
  },
  signatures: {
    type: Array,
    default: () => []
  },
  textAnnotations: {
    type: Array,
    default: () => []
  },
  selectedIndex: {
    type: Number,
    default: -1
  },
  selectedType: {
    type: String,
    default: null
  }
})

// Component lifecycle hooks can be added here if needed

const emit = defineEmits([
  'place-element',
  'drag-start',
  'select-element',
  'delete-signature',
  'delete-text',
  'edit-text',
  'container-click'
])

const pageCanvases = ref({})

const setPageRef = (el, pageNum) => {
  if (el) {
    pageCanvases.value[pageNum] = el
  }
}

const handleContainerClick = () => {
  emit('container-click')
}

// Expose refs to parent
// Note: We need to expose the actual value, not the ref wrapper
defineExpose({
  pageCanvases: pageCanvases,
  getCanvas: (pageNum) => pageCanvases.value[pageNum]
})
</script>

<style scoped>
.pdf-canvas-wrapper {
  flex: 1;
  overflow: hidden;
}

.pdf-scroll-container {
  width: 100%;
  height: 100%;
  overflow: auto;
  background: #e8eaed;
  padding: 2rem;
}

.pdf-container {
  transform-origin: top center;
  position: relative;
  margin: 0 auto;
  width: fit-content;
}

.pdf-page-wrapper {
  margin-bottom: 20px;
}

.pdf-page-wrapper:last-child {
  margin-bottom: 0;
}

.pdf-canvas {
  display: block;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  background: white;
}

.signature-overlay {
  position: absolute;
  cursor: grab;
  border: 2px dashed #4285f4;
  background: rgba(66, 133, 244, 0.1);
  will-change: transform;
  transform: translate3d(0, 0, 0);
  backface-visibility: hidden;
}

.signature-overlay:active {
  cursor: grabbing;
}

.signature-overlay:hover {
  border-color: #3367d6;
  background: rgba(66, 133, 244, 0.2);
}

.signature-overlay.selected {
  border-color: #1a73e8;
  border-width: 3px;
  border-style: solid;
  background: rgba(66, 133, 244, 0.15);
  box-shadow: 0 0 0 2px rgba(26, 115, 232, 0.3);
}

.text-overlay {
  position: absolute;
  cursor: grab;
  padding: 4px 8px;
  white-space: pre-wrap;
  word-break: break-word;
  user-select: none;
  border: 2px dashed transparent;
  transition: border-color 0.2s ease, background 0.2s ease;
  will-change: transform;
  transform: translate3d(0, 0, 0);
  backface-visibility: hidden;
}

.text-overlay:active {
  cursor: grabbing;
}

.text-overlay:hover {
  border-color: #4285f4;
  background: rgba(66, 133, 244, 0.1);
}

.text-overlay.selected {
  border-color: #1a73e8;
  border-width: 3px;
  border-style: solid;
  background: rgba(66, 133, 244, 0.15);
  box-shadow: 0 0 0 2px rgba(26, 115, 232, 0.3);
}

.delete-sig {
  position: absolute;
  top: -10px;
  right: -10px;
  width: 24px;
  height: 24px;
  background: #ea4335;
  color: white;
  border: none;
  border-radius: 50%;
  cursor: pointer;
  font-size: 18px;
  line-height: 1;
}

.delete-sig:hover {
  background: #c5221f;
}

@media (max-width: 768px) {
  .pdf-scroll-container {
    padding: 0.5rem;
    overflow-x: auto;
    overflow-y: auto;
  }

  .pdf-container {
    width: auto;
    max-width: none;
  }

  .pdf-page-wrapper {
    margin-bottom: 16px;
    width: auto;
  }

  .pdf-canvas {
    max-width: none;
  }

  .delete-sig {
    width: 28px;
    height: 28px;
    font-size: 20px;
    top: -8px;
    right: -8px;
  }
}

@media (max-width: 480px) {
  .pdf-scroll-container {
    padding: 0.5rem 0.25rem;
  }

  .pdf-page-wrapper {
    margin-bottom: 12px;
  }
}
</style>

