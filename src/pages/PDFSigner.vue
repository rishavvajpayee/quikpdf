<template>
  <div class="pdf-viewer-container">
    <!-- Menu Bar -->
    <EditorMenuBar
      :zoom="zoom"
      @new-document="newDocument"
      @save="exportPDFWithSignatures"
      @undo="undo"
      @redo="redo"
      @decrease-zoom="decreaseZoom"
      @increase-zoom="increaseZoom"
    />

    <!-- Toolbar with Upload and Signature buttons -->
    <div class="pdf-toolbar">
      <input
        type="file"
        ref="fileInput"
        @change="loadPDF"
        accept=".pdf"
        style="display: none"
      />
      <!-- <button @click="$refs.fileInput.click()" class="toolbar-btn">
        Load PDF
      </button> -->
      <button @click="createSignature" class="toolbar-btn" :disabled="!pdfLoaded">
        Add Signature
      </button>
      <button @click="clearSignatures" class="toolbar-btn" :disabled="signatures.length === 0">
        Clear Signatures
      </button>
    </div>

    <!-- Signature Placement Hint -->
    <div v-if="showPlacementHint" class="placement-hint">
      📍 Click anywhere on the PDF to place your signature
    </div>

    <div v-if="!pdfLoaded" class="pdf-not-loaded">
      <div class="empty-state">

        <h3>No PDF Document Loaded</h3>
        <p>Upload a PDF document to start adding signatures</p>
        <button @click="$refs.fileInput.click()" class="load-pdf-btn">
          Load PDF Document
        </button>
      </div>
    </div>

    <!-- PDF Canvas Container -->
    <div class="pdf-canvas-wrapper">
      <div class="pdf-scroll-container">
        <div class="pdf-container" :style="{ transform: `scale(${zoom})` }">
          <!-- Multiple page canvases -->
          <div
            v-for="pageNum in totalPages"
            :key="pageNum"
            class="pdf-page-wrapper"
            @click="placeSignature"
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
            :style="{
              left: sig.x + 'px',
              top: sig.y + 'px',
              width: sig.width + 'px',
              height: sig.height + 'px'
            }"
            @mousedown="startDrag($event, index)"
          >
            <img :src="sig.image" alt="Signature" style="width: 100%; height: 100%;" />
            <button class="delete-sig" @click="deleteSignature(index)">×</button>
          </div>
        </div>
      </div>
    </div>

    <!-- Export PDF Dialog -->
    <div v-if="showExportDialog" class="modal-overlay" @click="showExportDialog = false">
      <div class="modal-content" @click.stop>
        <h3>Save Signed PDF</h3>
        <div class="dialog-body">
          <label for="export-filename">File Name:</label>
          <input
            id="export-filename"
            type="text"
            v-model="exportFilename"
            class="filename-input"
            @keyup.enter="confirmExport"
          />
        </div>
        <div class="modal-actions">
          <button @click="showExportDialog = false" class="btn-secondary">Cancel</button>
          <button @click="confirmExport" class="btn-primary">Download</button>
        </div>
      </div>
    </div>

    <!-- Signature Drawing Modal -->
    <div v-if="showSignatureModal" class="modal-overlay" @click="closeSignatureModal">
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
        ></canvas>
        <div class="modal-actions">
          <button @click="clearSignatureCanvas" class="btn-secondary">Clear</button>
          <button @click="saveSignature" class="btn-primary">Add to PDF</button>
          <button @click="closeSignatureModal" class="btn-secondary">Cancel</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, watch } from 'vue'
import * as pdfjsLib from 'pdfjs-dist'
import { PDFDocument } from 'pdf-lib'
import EditorMenuBar from '../components/EditorMenuBar.vue'

// Set worker to local file
pdfjsLib.GlobalWorkerOptions.workerSrc = '/pdf.worker.min.mjs'

const zoom = ref(1)
const fileInput = ref(null)
const pdfLoaded = ref(false)
const totalPages = ref(0)
const pageCanvases = ref({})
const showPlacementHint = ref(false)
const showExportDialog = ref(false)
const exportFilename = ref('')
// Don't make PDF objects reactive - Vue reactivity breaks PDF.js internals
let pdfDocument = null
let originalPdfFile = null // Store original PDF file for export

const setPageRef = (el, pageNum) => {
  if (el) {
    pageCanvases.value[pageNum] = el
  }
}

const signatures = ref([])
const showSignatureModal = ref(false)
const signatureCanvas = ref(null)
const isDrawing = ref(false)
const signatureContext = ref(null)
const pendingSignature = ref(null)

// Dragging state
const dragging = ref(false)
const dragIndex = ref(-1)
const dragOffset = ref({ x: 0, y: 0 })

const loadPDF = async (event) => {
  const file = event.target.files[0]
  if (!file) return

  // Store the original file for later export
  originalPdfFile = file

  const fileReader = new FileReader()
  fileReader.onload = async (e) => {
    const typedarray = new Uint8Array(e.target.result)

    try {
      const loadingTask = pdfjsLib.getDocument({ data: typedarray })
      const pdf = await loadingTask.promise
      pdfDocument = pdf
      totalPages.value = pdf.numPages

      // Wait for Vue to create the canvas elements
      await new Promise(resolve => setTimeout(resolve, 100))

      // Render all pages at initial zoom
      for (let i = 1; i <= pdf.numPages; i++) {
        await renderPage(i, zoom.value)
      }

      pdfLoaded.value = true
    } catch (error) {
      console.error('Error loading PDF:', error)
      alert('Failed to load PDF. Please try another file.')
    }
  }
  fileReader.readAsArrayBuffer(file)
}

const renderPage = async (pageNum, currentZoom = zoom.value) => {
  if (!pdfDocument) return

  const canvas = pageCanvases.value[pageNum]
  if (!canvas) return

  const page = await pdfDocument.getPage(pageNum)
  
  // Use device pixel ratio for high-DPI displays (Retina, etc.)
  const pixelRatio = window.devicePixelRatio || 1
  // Adaptive base scale: lower at normal zoom, higher when zoomed in
  // This gives best quality at all zoom levels without over-rendering
  const baseScale = 2.0 // Balanced for quality and performance
  const scale = baseScale * pixelRatio * currentZoom
  
  const viewport = page.getViewport({ scale })
  const context = canvas.getContext('2d')

  // Set canvas internal resolution (actual pixels)
  canvas.width = viewport.width
  canvas.height = viewport.height
  
  // Set canvas display size (CSS pixels) - accounting for zoom
  canvas.style.width = `${viewport.width / (pixelRatio * currentZoom)}px`
  canvas.style.height = `${viewport.height / (pixelRatio * currentZoom)}px`

  const renderContext = {
    canvasContext: context,
    viewport: viewport
  }

  await page.render(renderContext).promise
}

// Watch zoom changes and re-render for crisp quality
let renderTimeout = null
watch(zoom, async (newZoom) => {
  if (!pdfDocument || !pdfLoaded.value) return
  
  // Debounce rendering to avoid too many re-renders while zooming
  clearTimeout(renderTimeout)
  renderTimeout = setTimeout(async () => {
    // Re-render all pages at new zoom level for crisp quality
    for (let i = 1; i <= totalPages.value; i++) {
      await renderPage(i, newZoom)
    }
  }, 100) // Reduced debounce for faster re-rendering
})

const createSignature = () => {
  showSignatureModal.value = true
  setTimeout(() => {
    if (signatureCanvas.value) {
      signatureContext.value = signatureCanvas.value.getContext('2d', { willReadFrequently: true })
      // High quality settings
      signatureContext.value.strokeStyle = '#000'
      signatureContext.value.lineWidth = 4 // Thicker line for better quality
      signatureContext.value.lineCap = 'round'
      signatureContext.value.lineJoin = 'round'
      signatureContext.value.imageSmoothingEnabled = true
      signatureContext.value.imageSmoothingQuality = 'high'
    }
  }, 10)
}

const closeSignatureModal = () => {
  showSignatureModal.value = false
  pendingSignature.value = null
}

const startDrawing = (e) => {
  isDrawing.value = true
  const rect = signatureCanvas.value.getBoundingClientRect()
  const canvas = signatureCanvas.value
  // Scale coordinates to match actual canvas resolution
  const scaleX = canvas.width / rect.width
  const scaleY = canvas.height / rect.height
  const x = (e.clientX - rect.left) * scaleX
  const y = (e.clientY - rect.top) * scaleY

  signatureContext.value.beginPath()
  signatureContext.value.moveTo(x, y)
}

const draw = (e) => {
  if (!isDrawing.value) return
  const rect = signatureCanvas.value.getBoundingClientRect()
  const canvas = signatureCanvas.value
  // Scale coordinates to match actual canvas resolution
  const scaleX = canvas.width / rect.width
  const scaleY = canvas.height / rect.height
  const x = (e.clientX - rect.left) * scaleX
  const y = (e.clientY - rect.top) * scaleY

  signatureContext.value.lineTo(x, y)
  signatureContext.value.stroke()
}

const stopDrawing = () => {
  isDrawing.value = false
}

const clearSignatureCanvas = () => {
  if (signatureContext.value && signatureCanvas.value) {
    signatureContext.value.clearRect(0, 0, signatureCanvas.value.width, signatureCanvas.value.height)
  }
}

const saveSignature = () => {
  if (!signatureCanvas.value) return

  // Save at maximum quality
  pendingSignature.value = signatureCanvas.value.toDataURL('image/png', 1.0)
  showSignatureModal.value = false

  // Show placement hint
  showPlacementHint.value = true
}

const placeSignature = (e) => {
  if (!pendingSignature.value) return
  if (dragging.value) return // Don't place signature while dragging

  const container = document.querySelector('.pdf-container').getBoundingClientRect()
  const x = (e.clientX - container.left) / zoom.value
  const y = (e.clientY - container.top) / zoom.value

  signatures.value.push({
    image: pendingSignature.value,
    x: x - 100,
    y: y - 37.5,
    width: 200,
    height: 75
  })

  pendingSignature.value = null
  showPlacementHint.value = false
}

const startDrag = (e, index) => {
  e.preventDefault()
  e.stopPropagation()

  dragging.value = true
  dragIndex.value = index

  const sig = signatures.value[index]
  const containerRect = document.querySelector('.pdf-container').getBoundingClientRect()

  // Calculate offset from mouse to signature top-left corner
  dragOffset.value = {
    x: (e.clientX - containerRect.left) / zoom.value - sig.x,
    y: (e.clientY - containerRect.top) / zoom.value - sig.y
  }

  document.addEventListener('mousemove', onDrag)
  document.addEventListener('mouseup', stopDrag)
}

const onDrag = (e) => {
  if (!dragging.value || dragIndex.value < 0) return

  e.preventDefault()

  // Get the PDF container position (not individual canvas)
  const container = document.querySelector('.pdf-container').getBoundingClientRect()
  const x = (e.clientX - container.left) / zoom.value - dragOffset.value.x
  const y = (e.clientY - container.top) / zoom.value - dragOffset.value.y

  signatures.value[dragIndex.value].x = x
  signatures.value[dragIndex.value].y = y
}

const stopDrag = (e) => {
  if (e) e.preventDefault()
  dragging.value = false
  dragIndex.value = -1
  document.removeEventListener('mousemove', onDrag)
  document.removeEventListener('mouseup', stopDrag)
}

const deleteSignature = (index) => {
  signatures.value.splice(index, 1)
}

const clearSignatures = () => {
  if (confirm('Clear all signatures?')) {
    signatures.value = []
  }
}

const exportPDFWithSignatures = async () => {
  if (!originalPdfFile || totalPages.value === 0) return

  // Get filename without extension and add _signed
  const originalName = originalPdfFile.name.replace(/\.pdf$/i, '')
  exportFilename.value = `${originalName}_signed`
  showExportDialog.value = true
}

const confirmExport = async () => {
  if (!exportFilename.value.trim()) {
    alert('Please enter a filename')
    return
  }

  showExportDialog.value = false

  try {
    // Read the original file fresh
    const arrayBuffer = await originalPdfFile.arrayBuffer()

    // Load the original PDF
    const pdfDoc = await PDFDocument.load(arrayBuffer)
    const pages = pdfDoc.getPages()

    // Process signatures for each page
    for (const sig of signatures.value) {
      // Convert signature image to PNG bytes
      const img = new Image()
      img.src = sig.image
      await new Promise((resolve) => { img.onload = resolve })

      // Create a high-resolution canvas for export
      const canvas = document.createElement('canvas')
      // Use 2x resolution for better quality
      const exportScale = 2
      canvas.width = sig.width * exportScale
      canvas.height = sig.height * exportScale
      const ctx = canvas.getContext('2d')
      ctx.imageSmoothingEnabled = true
      ctx.imageSmoothingQuality = 'high'
      ctx.drawImage(img, 0, 0, canvas.width, canvas.height)

      const pngDataUrl = canvas.toDataURL('image/png', 1.0)
      const pngBytes = await fetch(pngDataUrl).then(res => res.arrayBuffer())

      // Embed the signature image
      const pngImage = await pdfDoc.embedPng(pngBytes)

      // Calculate which page the signature is on
      let currentY = 0
      let targetPageIndex = 0
      let signatureYOnPage = sig.y
      const pixelRatio = window.devicePixelRatio || 1

      for (let i = 0; i < totalPages.value; i++) {
        const canvas = pageCanvases.value[i + 1]
        if (canvas) {
          // Use display height (CSS pixels), accounting for zoom
          const displayHeight = canvas.height / (pixelRatio * zoom.value)
          if (sig.y >= currentY && sig.y < currentY + displayHeight) {
            targetPageIndex = i
            signatureYOnPage = sig.y - currentY
            break
          }
          currentY += displayHeight + 20 // Account for gap between pages
        }
      }

      const page = pages[targetPageIndex]
      const pageHeight = page.getHeight()
      const pageWidth = page.getWidth()

      // Get the canvas dimensions for this page
      const pageCanvas = pageCanvases.value[targetPageIndex + 1]
      // Account for zoom when calculating canvas dimensions
      const canvasWidth = pageCanvas.width / (pixelRatio * zoom.value) // CSS pixels
      const canvasHeight = pageCanvas.height / (pixelRatio * zoom.value) // CSS pixels

      // Calculate scale factor
      const scaleX = pageWidth / canvasWidth
      const scaleY = pageHeight / canvasHeight

      // Convert canvas coordinates to PDF coordinates
      const pdfX = sig.x * scaleX
      const pdfWidth = sig.width * scaleX
      const pdfHeight = sig.height * scaleY

      // Convert Y coordinate (canvas uses top-left origin, PDF uses bottom-left)
      const pdfY = pageHeight - (signatureYOnPage * scaleY) - pdfHeight

      page.drawImage(pngImage, {
        x: pdfX,
        y: pdfY,
        width: pdfWidth,
        height: pdfHeight,
      })
    }

    // Save the PDF
    const pdfBytes = await pdfDoc.save()
    const blob = new Blob([pdfBytes], { type: 'application/pdf' })
    const url = URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url
    // Add .pdf extension if not present
    const filename = exportFilename.value.endsWith('.pdf')
      ? exportFilename.value
      : `${exportFilename.value}.pdf`
    a.download = filename
    a.click()
    URL.revokeObjectURL(url)
  } catch (error) {
    console.error('Error exporting PDF:', error)
    alert('Failed to export PDF. Please try again.')
  }
}

const newDocument = () => {
  if (confirm('This will clear the current PDF and signatures. Continue?')) {
    pdfDocument = null
    originalPdfFile = null
    pdfLoaded.value = false
    totalPages.value = 0
    pageCanvases.value = {}
    signatures.value = []
  }
}

const increaseZoom = () => {
  zoom.value = Math.min(zoom.value + 0.1, 2)
}

const decreaseZoom = () => {
  zoom.value = Math.max(zoom.value - 0.1, 0.5)
}

const undo = () => {
  if (signatures.value.length > 0) {
    signatures.value.pop()
  }
}

const redo = () => {
  // Could implement redo stack if needed
}
</script>

<style scoped>
.pdf-viewer-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  background-color: #f8f9fa;
}

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
  cursor: move;
  border: 2px dashed #4285f4;
  background: rgba(66, 133, 244, 0.1);
}

.signature-overlay:hover {
  border-color: #3367d6;
  background: rgba(66, 133, 244, 0.2);
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

.empty-state-icon {
  color: #5f6368;
  margin-bottom: 24px;
  opacity: 0.6;
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

.load-pdf-btn svg {
  width: 20px;
  height: 20px;
}
</style>

