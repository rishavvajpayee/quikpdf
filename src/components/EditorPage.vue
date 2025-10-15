<template>
  <div class="editor-workspace">
    <!-- Horizontal Ruler -->
    <div class="ruler horizontal">
      <div v-for="i in 21" :key="i" class="ruler-mark">
        {{ i - 1 }}
      </div>
    </div>

    <!-- Vertical Ruler -->
    <div class="ruler vertical">
      <div v-for="i in 30" :key="i" class="ruler-mark">
        {{ i - 1 }}
      </div>
    </div>

    <!-- Editor Content -->
    <div class="editor-scroll-container">
      <div class="page-container" :style="{ transform: `scale(${zoom})` }">
        <!-- Multiple pages container -->
        <div class="pages-wrapper">
          <div
            v-for="pageNum in pageCount"
            :key="pageNum"
            class="page a4"
            :class="{ 'first-page': pageNum === 1 }"
          >
            <!-- Margin lines removed -->
            <div v-if="pageNum === 1" class="editor-content">
              <editor-content :editor="editor" />
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, watch, nextTick } from 'vue'
import { EditorContent } from '@tiptap/vue-3'

const props = defineProps({
  editor: Object,
  zoom: Number
})

const pageCount = ref(1)

// Watch for content changes and calculate if we need more pages
watch(() => props.editor?.state.doc.content.size, async () => {
  await nextTick()
  checkPageOverflow()
}, { immediate: true })

const checkPageOverflow = () => {
  const editorContent = document.querySelector('.editor-content')
  if (!editorContent) return

  const contentHeight = editorContent.scrollHeight
  const pageHeight = 297 // A4 height in mm
  const marginHeight = 3 // 1.5cm top + 1.5cm bottom = 3cm
  const usableHeight = pageHeight - marginHeight

  // Convert mm to pixels (approximate: 1mm ≈ 3.78px at 96dpi)
  const usableHeightPx = usableHeight * 3.78

  const neededPages = Math.ceil(contentHeight / usableHeightPx)
  pageCount.value = Math.max(1, neededPages)
}
</script>

<style scoped>
.editor-workspace {
  flex: 1;
  display: grid;
  grid-template-columns: 30px 1fr;
  grid-template-rows: 30px 1fr;
  position: relative;
  overflow: hidden;
}

.ruler {
  background: #f1f3f4;
  border: 1px solid #e0e0e0;
  display: flex;
  color: #5f6368;
  font-size: 10px;
}

.ruler.horizontal {
  grid-column: 2;
  grid-row: 1;
  flex-direction: row;
}

.ruler.vertical {
  grid-column: 1;
  grid-row: 2;
  flex-direction: column;
}

.ruler-mark {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
}

.editor-scroll-container {
  grid-column: 2;
  grid-row: 2;
  overflow: auto;
  background-color: #e8eaed;
  padding: 2rem;
}

.page-container {
  transform-origin: top center;
  padding: 20px;
}

.pages-wrapper {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.page.a4 {
  width: 210mm;
  height: 297mm;
  background: white;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  margin: 0 auto;
  position: relative;
  page-break-after: always;
  break-after: page;
}

.page.a4.first-page .editor-content {
  overflow: visible;
}

.margin-lines {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  pointer-events: none;
  z-index: 0;
}

.margin-line {
  position: absolute;
  background-color: rgba(0,0,0,0.1);
  pointer-events: none;
}

.margin-line.top {
  top: 1.5cm;
  left: 0;
  right: 0;
  height: 1px;
}

.margin-line.right {
  top: 0;
  right: 1.5cm;
  bottom: 0;
  width: 1px;
}

.margin-line.bottom {
  bottom: 1.5cm;
  left: 0;
  right: 0;
  height: 1px;
}

.margin-line.left {
  top: 0;
  left: 1.5cm;
  bottom: 0;
  width: 1px;
}

.editor-content {
  padding: 1.5cm;
  height: calc(297mm - 3cm);
  position: relative;
  z-index: 1;
  overflow: visible;
}

.editor-content * {
  outline: none !important;
  box-shadow: none !important;
}

.editor-content *:focus {
  outline: none !important;
  box-shadow: none !important;
}

/* CSS to help with page breaks */
:deep(.ProseMirror) {
  overflow: visible;
}
</style>
