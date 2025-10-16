<template>
  <div class="pdf-viewer-container">
    <!-- Combined Header with Logo and Menu -->
    <div class="app-header">
      <div class="logo">
        <svg width="32" height="32" viewBox="0 0 32 32" fill="none" xmlns="http://www.w3.org/2000/svg">
          <rect width="32" height="32" rx="6" fill="#4285f4"/>
          <circle cx="16" cy="15" r="7" stroke="white" stroke-width="2.5" fill="none"/>
          <line x1="20" y1="20" x2="23" y2="23" stroke="white" stroke-width="2.5" stroke-linecap="round"/>
        </svg>
        <span class="logo-text">quikPDF</span>
      </div>

      <EditorMenuBar
        :zoom="zoom"
        @new-document="newDocument"
        @save="exportPDFWithSignatures"
        @undo="undo"
        @redo="redo"
        @decrease-zoom="decreaseZoom"
        @increase-zoom="increaseZoom"
      />
    </div>

    <!-- Toolbar with Upload and Signature buttons -->
    <PDFToolbar
      ref="toolbarRef"
      :pdf-loaded="pdfLoaded"
      :has-annotations="signatures.length > 0 || textAnnotations.length > 0"
      @add-signature="createSignature"
      @add-text="createText"
      @clear-all="clearSignatures"
      @file-selected="loadPDF"
    />

    <!-- Placement Hint -->
    <PlacementHint
      :show="showPlacementHint"
      :type="pendingSignature ? 'signature' : 'text'"
    />

    <!-- Empty State -->
    <EmptyState
      :show="!pdfLoaded"
      @load-pdf="toolbarRef?.clickFileInput()"
    />

    <!-- PDF Canvas Container -->
    <PDFCanvas
      v-if="pdfLoaded"
      ref="pdfCanvasRef"
      :zoom="zoom"
      :total-pages="totalPages"
      :signatures="signatures"
      :text-annotations="textAnnotations"
      :selected-index="selectedIndex"
      :selected-type="selectedType"
      @place-element="placeSignature"
      @drag-start="startDrag"
      @select-element="selectElement"
      @delete-signature="deleteSignature"
      @delete-text="deleteText"
      @edit-text="editText"
      @container-click="handleContainerClick"
    />

    <!-- Export PDF Dialog -->
    <ExportDialog
      :show="showExportDialog"
      :filename="exportFilename"
      @close="showExportDialog = false"
      @confirm="confirmExport"
    />

    <!-- Signature Drawing Modal -->
    <SignatureModal
      :show="showSignatureModal"
      @close="closeSignatureModal"
      @save="saveSignature"
    />

    <!-- Text Input Modal -->
    <TextModal
      :show="showTextModal"
      :is-editing="editingTextIndex !== null"
      :content="newTextContent"
      :font-size="textFontSize"
      :color="textColor"
      :bold="textBold"
      :italic="textItalic"
      @close="closeTextModal"
      @save="saveText"
    />
  </div>
</template>

<script setup>
import { ref, watch, onMounted, onUnmounted, nextTick } from 'vue'
import * as pdfjsLib from 'pdfjs-dist'
import { PDFDocument, rgb } from 'pdf-lib'
import EditorMenuBar from '../components/EditorMenuBar.vue'
import PDFToolbar from '../components/pdf/PDFToolbar.vue'
import PlacementHint from '../components/pdf/PlacementHint.vue'
import EmptyState from '../components/pdf/EmptyState.vue'
import PDFCanvas from '../components/pdf/PDFCanvas.vue'
import ExportDialog from '../components/pdf/ExportDialog.vue'
import SignatureModal from '../components/pdf/SignatureModal.vue'
import TextModal from '../components/pdf/TextModal.vue'

// Set worker to local file
pdfjsLib.GlobalWorkerOptions.workerSrc = '/pdf.worker.min.mjs'

// Refs
const toolbarRef = ref(null)
const pdfCanvasRef = ref(null)
const zoom = ref(1)
const pdfLoaded = ref(false)
const totalPages = ref(0)
const showPlacementHint = ref(false)
const showExportDialog = ref(false)
const exportFilename = ref('')

// Don't make PDF objects reactive - Vue reactivity breaks PDF.js internals
let pdfDocument = null
let originalPdfFile = null // Store original PDF file for export

const signatures = ref([])
const showSignatureModal = ref(false)
const pendingSignature = ref(null)

// Text annotation state
const textAnnotations = ref([])
const showTextModal = ref(false)
const newTextContent = ref('')
const textFontSize = ref(16)
const textColor = ref('#000000')
const textBold = ref(false)
const textItalic = ref(false)
const pendingText = ref(null)
const editingTextIndex = ref(null)

// Dragging state
const dragging = ref(false)
const dragIndex = ref(-1)
const dragType = ref('signature') // 'signature' or 'text'
const dragOffset = ref({ x: 0, y: 0 })
let dragContainerRect = null // Cache container rect during drag
let animationFrameId = null

// Selection and copy-paste state
const selectedIndex = ref(-1)
const selectedType = ref(null) // 'signature' or 'text'
const copiedElement = ref(null)

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
      pdfLoaded.value = true

      // Wait for Vue to create the PDFCanvas component and canvas elements
      await nextTick()
      
      // Wait for canvas refs to be properly set with retry logic
      let retries = 0
      const maxRetries = 10
      while (retries < maxRetries) {
        await new Promise(resolve => setTimeout(resolve, 50))
        // Try to get the first canvas to verify refs are ready
        const testCanvas = pdfCanvasRef.value?.getCanvas(1)
        if (testCanvas) {
          break
        }
        retries++
      }

      // Render all pages at initial zoom
      for (let i = 1; i <= pdf.numPages; i++) {
        await renderPage(i, zoom.value)
      }
    } catch (error) {
      console.error('Error loading PDF:', error)
      alert('Failed to load PDF. Please try another file.')
    }
  }
  fileReader.readAsArrayBuffer(file)
}

const renderPage = async (pageNum, currentZoom = zoom.value) => {
  if (!pdfDocument) return
  if (!pdfCanvasRef.value) return

  const canvas = pdfCanvasRef.value.getCanvas(pageNum)
  if (!canvas) return

  const page = await pdfDocument.getPage(pageNum)

  // Use device pixel ratio for high-DPI displays (Retina, etc.)
  const pixelRatio = window.devicePixelRatio || 1
  
  // Use a consistent base scale for quality
  const baseScale = 1.5
  const scale = baseScale * currentZoom

  const viewport = page.getViewport({ scale })
  const context = canvas.getContext('2d')

  // Set canvas internal resolution (actual pixels) with device pixel ratio
  canvas.width = viewport.width * pixelRatio
  canvas.height = viewport.height * pixelRatio

  // Set canvas display size (CSS pixels)
  canvas.style.width = `${viewport.width}px`
  canvas.style.height = `${viewport.height}px`

  // Scale the context to match device pixel ratio
  context.scale(pixelRatio, pixelRatio)

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
}

const closeSignatureModal = () => {
  showSignatureModal.value = false
  pendingSignature.value = null
}

const saveSignature = (dataUrl) => {
  pendingSignature.value = dataUrl
  showSignatureModal.value = false
  showPlacementHint.value = true
}

const placeSignature = (e) => {
  if (!pendingSignature.value && !pendingText.value) return
  if (dragging.value) return // Don't place signature while dragging

  const container = document.querySelector('.pdf-container').getBoundingClientRect()
  
  // Handle touch events
  const clientX = e.changedTouches ? e.changedTouches[0].clientX : e.clientX
  const clientY = e.changedTouches ? e.changedTouches[0].clientY : e.clientY
  
  const x = (clientX - container.left) / zoom.value
  const y = (clientY - container.top) / zoom.value

  console.log('Placing signature:', {
    click: { clientX, clientY },
    container: { left: container.left, top: container.top },
    zoom: zoom.value,
    calculated: { x, y }
  })

  if (pendingSignature.value) {
    const sig = {
      image: pendingSignature.value,
      x: x - 100,
      y: y - 37.5,
      width: 200,
      height: 75
    }
    console.log('Signature stored:', sig)
    signatures.value.push(sig)
    pendingSignature.value = null
  } else if (pendingText.value) {
    textAnnotations.value.push({
      ...pendingText.value,
      x: x - 50,
      y: y - 10
    })
    pendingText.value = null
  }

  showPlacementHint.value = false
}

const startDrag = (e, index, type) => {
  e.preventDefault()
  e.stopPropagation()

  dragging.value = true
  dragIndex.value = index
  dragType.value = type

  const item = type === 'signature' ? signatures.value[index] : textAnnotations.value[index]

  // Cache the container rect once at the start of drag
  dragContainerRect = document.querySelector('.pdf-container').getBoundingClientRect()

  // Handle touch events
  const clientX = e.touches ? e.touches[0].clientX : e.clientX
  const clientY = e.touches ? e.touches[0].clientY : e.clientY

  // Calculate offset from mouse/touch to item top-left corner
  dragOffset.value = {
    x: (clientX - dragContainerRect.left) / zoom.value - item.x,
    y: (clientY - dragContainerRect.top) / zoom.value - item.y
  }

  // Add grabbing cursor to body during drag
  document.body.style.cursor = 'grabbing'
  document.body.style.userSelect = 'none'

  document.addEventListener('mousemove', onDrag, { passive: false })
  document.addEventListener('mouseup', stopDrag)
  document.addEventListener('touchmove', onDrag, { passive: false })
  document.addEventListener('touchend', stopDrag)
}

const onDrag = (e) => {
  if (!dragging.value || dragIndex.value < 0) return

  e.preventDefault()

  // Cancel any pending animation frame
  if (animationFrameId) {
    cancelAnimationFrame(animationFrameId)
  }

  // Handle touch events
  const clientX = e.touches ? e.touches[0].clientX : e.clientX
  const clientY = e.touches ? e.touches[0].clientY : e.clientY

  // Use requestAnimationFrame for smooth updates
  animationFrameId = requestAnimationFrame(() => {
    // Use cached container rect for better performance
    const x = (clientX - dragContainerRect.left) / zoom.value - dragOffset.value.x
    const y = (clientY - dragContainerRect.top) / zoom.value - dragOffset.value.y

    if (dragType.value === 'signature') {
      signatures.value[dragIndex.value].x = x
      signatures.value[dragIndex.value].y = y
    } else if (dragType.value === 'text') {
      textAnnotations.value[dragIndex.value].x = x
      textAnnotations.value[dragIndex.value].y = y
    }
  })
}

const stopDrag = (e) => {
  if (e) e.preventDefault()

  // Cancel any pending animation frame
  if (animationFrameId) {
    cancelAnimationFrame(animationFrameId)
    animationFrameId = null
  }

  dragging.value = false
  dragIndex.value = -1
  dragContainerRect = null

  // Restore cursor and user-select
  document.body.style.cursor = ''
  document.body.style.userSelect = ''

  document.removeEventListener('mousemove', onDrag)
  document.removeEventListener('mouseup', stopDrag)
  document.removeEventListener('touchmove', onDrag)
  document.removeEventListener('touchend', stopDrag)
}

const deleteSignature = (index) => {
  signatures.value.splice(index, 1)
}

const deleteText = (index) => {
  textAnnotations.value.splice(index, 1)
}

const clearSignatures = () => {
  if (confirm('Clear all signatures and text annotations?')) {
    signatures.value = []
    textAnnotations.value = []
  }
}

// Text annotation functions
const createText = () => {
  newTextContent.value = ''
  textFontSize.value = 16
  textColor.value = '#000000'
  textBold.value = false
  textItalic.value = false
  editingTextIndex.value = null
  showTextModal.value = true
}

const closeTextModal = () => {
  showTextModal.value = false
  editingTextIndex.value = null
  pendingText.value = null
}

const saveText = (textData) => {
  if (!textData.content.trim()) return

  if (editingTextIndex.value !== null) {
    // Update existing text
    const textItem = textAnnotations.value[editingTextIndex.value]
    textItem.content = textData.content
    textItem.fontSize = textData.fontSize
    textItem.color = textData.color
    textItem.bold = textData.bold
    textItem.italic = textData.italic
    editingTextIndex.value = null
  } else {
    // Create new text for placement
    pendingText.value = textData
    showPlacementHint.value = true
  }

  showTextModal.value = false
}

const editText = (index) => {
  const text = textAnnotations.value[index]
  newTextContent.value = text.content
  textFontSize.value = text.fontSize
  textColor.value = text.color
  textBold.value = text.bold
  textItalic.value = text.italic
  editingTextIndex.value = index
  showTextModal.value = true
}

const exportPDFWithSignatures = async () => {
  if (!originalPdfFile || totalPages.value === 0) return

  // Get filename without extension and add _signed
  const originalName = originalPdfFile.name.replace(/\.pdf$/i, '')
  exportFilename.value = `${originalName}_signed`
  showExportDialog.value = true
}

const confirmExport = async (filename) => {
  if (!filename.trim()) {
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

    // Embed fonts for text annotations
    const helveticaFont = await pdfDoc.embedFont('Helvetica')
    const helveticaBoldFont = await pdfDoc.embedFont('Helvetica-Bold')
    const helveticaObliqueFont = await pdfDoc.embedFont('Helvetica-Oblique')
    const helveticaBoldObliqueFont = await pdfDoc.embedFont('Helvetica-BoldOblique')

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
        const canvas = pdfCanvasRef.value?.getCanvas(i + 1)
        if (canvas) {
          // Get display height from CSS pixels (unscaled by container transform)
          const displayHeight = parseFloat(canvas.style.height)
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

      // Get the canvas dimensions for this page (CSS pixels, unscaled)
      const pageCanvas = pdfCanvasRef.value?.getCanvas(targetPageIndex + 1)
      const canvasWidth = parseFloat(pageCanvas.style.width)
      const canvasHeight = parseFloat(pageCanvas.style.height)

      console.log('Export signature:', {
        sigPosition: { x: sig.x, y: sig.y, width: sig.width, height: sig.height },
        targetPage: targetPageIndex,
        canvasDims: { width: canvasWidth, height: canvasHeight },
        pageDims: { width: pageWidth, height: pageHeight }
      })

      // Calculate scale factor from canvas (CSS pixels) to PDF coordinates
      const scaleX = pageWidth / canvasWidth
      const scaleY = pageHeight / canvasHeight

      // Convert canvas coordinates to PDF coordinates
      const pdfX = sig.x * scaleX
      const pdfWidth = sig.width * scaleX
      const pdfHeight = sig.height * scaleY

      // Convert Y coordinate (canvas uses top-left origin, PDF uses bottom-left)
      const pdfY = pageHeight - (signatureYOnPage * scaleY) - pdfHeight

      console.log('PDF coordinates:', { pdfX, pdfY, pdfWidth, pdfHeight, scaleX, scaleY })

      page.drawImage(pngImage, {
        x: pdfX,
        y: pdfY,
        width: pdfWidth,
        height: pdfHeight,
      })
    }

    // Process text annotations for each page
    for (const textItem of textAnnotations.value) {
      // Calculate which page the text is on
      let currentY = 0
      let targetPageIndex = 0
      let textYOnPage = textItem.y
      const pixelRatio = window.devicePixelRatio || 1

      for (let i = 0; i < totalPages.value; i++) {
        const canvas = pdfCanvasRef.value?.getCanvas(i + 1)
        if (canvas) {
          // Get display height from CSS pixels
          const displayHeight = parseFloat(canvas.style.height) || canvas.height / pixelRatio
          if (textItem.y >= currentY && textItem.y < currentY + displayHeight) {
            targetPageIndex = i
            textYOnPage = textItem.y - currentY
            break
          }
          currentY += displayHeight + 20 // Account for gap between pages
        }
      }

      const page = pages[targetPageIndex]
      const pageHeight = page.getHeight()
      const pageWidth = page.getWidth()

      // Get the canvas dimensions for this page (CSS pixels)
      const pageCanvas = pdfCanvasRef.value?.getCanvas(targetPageIndex + 1)
      const canvasWidth = parseFloat(pageCanvas.style.width) || pageCanvas.width / pixelRatio
      const canvasHeight = parseFloat(pageCanvas.style.height) || pageCanvas.height / pixelRatio

      // Calculate scale factor
      const scaleX = pageWidth / canvasWidth
      const scaleY = pageHeight / canvasHeight

      // Convert canvas coordinates to PDF coordinates
      // Add 8px horizontal padding from text-overlay CSS
      const pdfX = (textItem.x + 8) * scaleX
      const pdfFontSize = textItem.fontSize * scaleY

      // Convert Y coordinate (canvas uses top-left origin, PDF uses bottom-left)
      // Text baseline in browser is at: Y + padding-top + line-height
      // PDF drawText positions at the baseline
      // Adding a bit extra to account for line-height (font size * 1.2 is typical)
      const paddingTop = 4
      const paddingBottom = 4
      const textY = textYOnPage + paddingTop + paddingBottom + textItem.fontSize
      const pdfY = pageHeight - (textY * scaleY)

      // Convert hex color to RGB values
      const hexToRgb = (hex) => {
        const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex)
        return result ? {
          r: parseInt(result[1], 16) / 255,
          g: parseInt(result[2], 16) / 255,
          b: parseInt(result[3], 16) / 255
        } : { r: 0, g: 0, b: 0 }
      }

      const rgbValues = hexToRgb(textItem.color)

      // Select the appropriate font based on bold and italic
      let font = helveticaFont
      if (textItem.bold && textItem.italic) {
        font = helveticaBoldObliqueFont
      } else if (textItem.bold) {
        font = helveticaBoldFont
      } else if (textItem.italic) {
        font = helveticaObliqueFont
      }

      page.drawText(textItem.content, {
        x: pdfX,
        y: pdfY,
        size: pdfFontSize,
        font: font,
        color: rgb(rgbValues.r, rgbValues.g, rgbValues.b),
      })
    }

    // Save the PDF
    const pdfBytes = await pdfDoc.save()
    const blob = new Blob([pdfBytes], { type: 'application/pdf' })
    const url = URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url
    // Add .pdf extension if not present
    const finalFilename = filename.endsWith('.pdf')
      ? filename
      : `${filename}.pdf`
    a.download = finalFilename
    a.click()
    URL.revokeObjectURL(url)
  } catch (error) {
    console.error('Error exporting PDF:', error)
    alert('Failed to export PDF. Please try again.')
  }
}

const newDocument = () => {
  if (confirm('This will clear the current PDF and all annotations. Continue?')) {
    pdfDocument = null
    originalPdfFile = null
    pdfLoaded.value = false
    totalPages.value = 0
    signatures.value = []
    textAnnotations.value = []
  }
}

const increaseZoom = () => {
  zoom.value = Math.min(zoom.value + 0.1, 2)
}

const decreaseZoom = () => {
  zoom.value = Math.max(zoom.value - 0.1, 0.5)
}

const undo = () => {
  // Remove the most recently added item (signature or text)
  if (textAnnotations.value.length > 0 && signatures.value.length === 0) {
    textAnnotations.value.pop()
  } else if (signatures.value.length > 0 && textAnnotations.value.length === 0) {
    signatures.value.pop()
  } else if (textAnnotations.value.length > 0 && signatures.value.length > 0) {
    // Remove whichever was added most recently (just remove from signatures for simplicity)
    signatures.value.pop()
  }
}

const redo = () => {
  // Could implement redo stack if needed
}

// Selection and copy-paste functions
const selectElement = (index, type, event) => {
  if (event) {
    event.stopPropagation()
  }
  selectedIndex.value = index
  selectedType.value = type
}

const handleContainerClick = () => {
  // Deselect when clicking on empty space
  if (!pendingSignature.value && !pendingText.value) {
    selectedIndex.value = -1
    selectedType.value = null
  }
}

const copyElement = () => {
  if (selectedIndex.value === -1 || !selectedType.value) return

  if (selectedType.value === 'signature') {
    copiedElement.value = {
      type: 'signature',
      data: { ...signatures.value[selectedIndex.value] }
    }
  } else if (selectedType.value === 'text') {
    copiedElement.value = {
      type: 'text',
      data: { ...textAnnotations.value[selectedIndex.value] }
    }
  }
}

const pasteElement = () => {
  if (!copiedElement.value) return

  if (copiedElement.value.type === 'signature') {
    signatures.value.push({
      ...copiedElement.value.data,
      x: copiedElement.value.data.x + 20, // Offset slightly
      y: copiedElement.value.data.y + 20
    })
    // Select the newly pasted element
    selectedIndex.value = signatures.value.length - 1
    selectedType.value = 'signature'
  } else if (copiedElement.value.type === 'text') {
    textAnnotations.value.push({
      ...copiedElement.value.data,
      x: copiedElement.value.data.x + 20, // Offset slightly
      y: copiedElement.value.data.y + 20
    })
    // Select the newly pasted element
    selectedIndex.value = textAnnotations.value.length - 1
    selectedType.value = 'text'
  }
}

// Keyboard event handler
const handleKeyboard = (e) => {
  // Copy: Ctrl+C or Cmd+C
  if ((e.ctrlKey || e.metaKey) && e.key === 'c') {
    e.preventDefault()
    copyElement()
  }
  // Paste: Ctrl+V or Cmd+V
  else if ((e.ctrlKey || e.metaKey) && e.key === 'v') {
    e.preventDefault()
    pasteElement()
  }
  // Delete: Delete or Backspace
  else if (e.key === 'Delete' || e.key === 'Backspace') {
    if (selectedIndex.value !== -1 && selectedType.value) {
      e.preventDefault()
      if (selectedType.value === 'signature') {
        deleteSignature(selectedIndex.value)
      } else if (selectedType.value === 'text') {
        deleteText(selectedIndex.value)
      }
      selectedIndex.value = -1
      selectedType.value = null
    }
  }
  // Deselect: Escape
  else if (e.key === 'Escape') {
    selectedIndex.value = -1
    selectedType.value = null
  }
}

// Handle window resize for mobile rotation
let resizeTimeout = null
const handleResize = () => {
  if (!pdfDocument || !pdfLoaded.value) return
  
  clearTimeout(resizeTimeout)
  resizeTimeout = setTimeout(async () => {
    // Re-render all pages at new viewport size
    for (let i = 1; i <= totalPages.value; i++) {
      await renderPage(i, zoom.value)
    }
  }, 300)
}

// Add keyboard event listener on mount
onMounted(() => {
  document.addEventListener('keydown', handleKeyboard)
  window.addEventListener('resize', handleResize)
})

onUnmounted(() => {
  document.removeEventListener('keydown', handleKeyboard)
  window.removeEventListener('resize', handleResize)
})
</script>

<style scoped>
.pdf-viewer-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  background-color: #f8f9fa;
}

.app-header {
  display: flex;
  align-items: center;
  background: #ffffff;
  padding: 4px 24px;
  border-bottom: 1px solid #e0e0e0;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
  gap: 16px;
}

.logo {
  display: flex;
  align-items: center;
  gap: 12px;
}

.logo-text {
  font-size: 20px;
  font-weight: 600;
  color: #202124;
  letter-spacing: -0.5px;
  white-space: nowrap;
}

@media (max-width: 768px) {
  .app-header {
    padding: 8px 12px;
    gap: 8px;
    flex-wrap: wrap;
  }

  .logo {
    gap: 8px;
  }

  .logo svg {
    width: 28px;
    height: 28px;
  }

  .logo-text {
    font-size: 18px;
  }
}

@media (max-width: 480px) {
  .app-header {
    padding: 6px 10px;
  }

  .logo svg {
    width: 24px;
    height: 24px;
  }

  .logo-text {
    font-size: 16px;
  }
}
</style>
