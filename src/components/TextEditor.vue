<template>
  <div class="editor-container">
    <!-- Menu Bar Component -->
    <EditorMenuBar
      :zoom="zoom"
      @new-document="newDocument"
      @save="save"
      @undo="undo"
      @redo="redo"
      @decrease-zoom="decreaseZoom"
      @increase-zoom="increaseZoom"
    />

    <!-- Toolbar Component -->
    <EditorToolbar
      :is-bold="isBold"
      :is-italic="isItalic"
      :is-underline="isUnderline"
      :is-highlight="isHighlight"
      :is-bullet-list="editor?.isActive('bulletList')"
      :is-ordered-list="editor?.isActive('orderedList')"
      :alignment="alignment"
      @set-heading="setHeading"
      @toggle-bold="toggleBold"
      @toggle-italic="toggleItalic"
      @toggle-underline="toggleUnderline"
      @toggle-highlight="toggleHighlight"
      @toggle-bullet-list="toggleBulletList"
      @toggle-ordered-list="toggleOrderedList"
      @set-alignment="setTextAlign"
    />

    <!-- Editor Page Component -->
    <EditorPage
      :editor="editor"
      :zoom="zoom"
    />

    <!-- Save Dialog Component -->
    <SaveDialog
      :show="showSaveDialog"
      v-model:filename="filename"
      @close="closeSaveDialog"
      @download="downloadPDF"
    />
  </div>
</template>

<script setup>
import { ref, computed, onBeforeUnmount } from 'vue'
import { useEditor } from '@tiptap/vue-3'
import StarterKit from '@tiptap/starter-kit'
import Underline from '@tiptap/extension-underline'
import TextAlign from '@tiptap/extension-text-align'
import Highlight from '@tiptap/extension-highlight'
import Placeholder from '@tiptap/extension-placeholder'
import html2pdf from 'html2pdf.js'

import EditorMenuBar from './EditorMenuBar.vue'
import EditorToolbar from './EditorToolbar.vue'
import EditorPage from './EditorPage.vue'
import SaveDialog from './SaveDialog.vue'

const alignment = ref('left')
const zoom = ref(1)
const showSaveDialog = ref(false)
const filename = ref('document')

const editor = useEditor({
  content: '<p></p>',
  extensions: [
    StarterKit.configure({
      bulletList: {
        keepMarks: true,
        keepAttributes: false,
        HTMLAttributes: {
          class: 'tiptap-bullet-list',
        },
      },
      orderedList: {
        keepMarks: true,
        keepAttributes: false,
        HTMLAttributes: {
          class: 'tiptap-ordered-list',
        },
      },
      listItem: {
        HTMLAttributes: {
          class: 'tiptap-list-item',
        },
      },
    }),
    Underline,
    Highlight,
    TextAlign.configure({
      types: ['heading', 'paragraph', 'listItem'],
    }),
    Placeholder.configure({
      placeholder: 'Start typing...',
    }),
  ],
  editorProps: {
    attributes: {
      class: 'prose focus:outline-none',
    },
  },
  autofocus: true,
})

const isBold = computed(() => editor.value?.isActive('bold'))
const isItalic = computed(() => editor.value?.isActive('italic'))
const isUnderline = computed(() => editor.value?.isActive('underline'))
const isHighlight = computed(() => editor.value?.isActive('highlight'))

const toggleBold = () => {
  editor.value?.chain().focus().toggleBold().run()
}

const toggleItalic = () => {
  editor.value?.chain().focus().toggleItalic().run()
}

const toggleUnderline = () => {
  editor.value?.chain().focus().toggleUnderline().run()
}

const toggleHighlight = () => {
  editor.value?.chain().focus().toggleHighlight().run()
}

const toggleBulletList = () => {
  editor.value?.chain().focus().toggleBulletList().run()
}

const toggleOrderedList = () => {
  editor.value?.chain().focus().toggleOrderedList().run()
}

const undo = () => {
  editor.value?.chain().focus().undo().run()
}

const redo = () => {
  editor.value?.chain().focus().redo().run()
}

const increaseZoom = () => {
  zoom.value = Math.min(zoom.value + 0.1, 2)
}

const decreaseZoom = () => {
  zoom.value = Math.max(zoom.value - 0.1, 0.5)
}

const setTextAlign = (event) => {
  alignment.value = event.target.value
  editor.value?.chain().focus().setTextAlign(alignment.value).run()
}

const setHeading = (event) => {
  const level = event.target.value
  if (level === 'paragraph') {
    editor.value?.chain().focus().setParagraph().run()
  } else {
    editor.value?.chain().focus().setHeading({ level: parseInt(level) }).run()
  }
}

const newDocument = () => {
  editor.value?.commands.setContent('<p></p>')
  editor.value?.commands.focus()
}

const save = () => {
  showSaveDialog.value = true
}

const closeSaveDialog = () => {
  showSaveDialog.value = false
}

const downloadPDF = () => {
  const content = editor.value?.getHTML()

  // Create a wrapper with proper A4 dimensions
  const wrapper = document.createElement('div')
  wrapper.style.cssText = `
    width: 210mm;
    min-height: 297mm;
    background: white;
    box-sizing: border-box;
    padding: 1.5cm;
  `

  // Create content container
  const element = document.createElement('div')
  element.innerHTML = content
  element.style.cssText = `
    font-family: Arial, sans-serif;
    font-size: 11pt;
    line-height: 1.5;
    color: #000;
  `

  // Add comprehensive styles
  const styleElement = document.createElement('style')
  styleElement.textContent = `
    * { box-sizing: border-box; }

    h1 {
      font-size: 24pt;
      font-weight: bold;
      margin: 0 0 12pt 0;
      line-height: 1.3;
    }
    h2 {
      font-size: 20pt;
      font-weight: bold;
      margin: 0 0 12pt 0;
      line-height: 1.3;
    }
    h3 {
      font-size: 16pt;
      font-weight: bold;
      margin: 0 0 10pt 0;
      line-height: 1.3;
    }
    h4 {
      font-size: 14pt;
      font-weight: bold;
      margin: 0 0 10pt 0;
      line-height: 1.3;
    }
    h5 {
      font-size: 12pt;
      font-weight: bold;
      margin: 0 0 10pt 0;
      line-height: 1.3;
    }
    h6 {
      font-size: 11pt;
      font-weight: bold;
      margin: 0 0 10pt 0;
      line-height: 1.3;
    }
    p {
      margin: 0 0 10pt 0;
      min-height: 1em;
      line-height: 1.5;
      font-size: 11pt;
    }
    p:first-child { margin-top: 0; }
    strong, b { font-weight: bold; }
    em, i { font-style: italic; }
    u { text-decoration: underline; }
    mark { background-color: yellow; padding: 2px; }

    /* Text alignment */
    [style*="text-align: left"] { text-align: left; }
    [style*="text-align: center"] { text-align: center; }
    [style*="text-align: right"] { text-align: right; }
    [style*="text-align: justify"] { text-align: justify; }

    .has-text-align-left { text-align: left; }
    .has-text-align-center { text-align: center; }
    .has-text-align-right { text-align: right; }
    .has-text-align-justify { text-align: justify; }

    /* List styles for PDF */
    ul, ol {
      padding-left: 30px;
      margin: 0 0 10pt 0;
    }

    li {
      margin: 0 0 2pt 0;
      padding-left: 5px;
    }

    li p {
      margin: 0;
    }

    /* Nested numbering */
    ol { list-style-type: decimal; }
    ol ol { list-style-type: lower-alpha; }
    ol ol ol { list-style-type: lower-roman; }

    /* Nested bullets */
    ul { list-style-type: disc; }
    ul ul { list-style-type: circle; }
    ul ul ul { list-style-type: square; }
  `

  wrapper.appendChild(styleElement)
  wrapper.appendChild(element)

  const opt = {
    margin: 0,
    filename: `${filename.value || 'document'}.pdf`,
    image: { type: 'jpeg', quality: 0.98 },
    html2canvas: {
      scale: 2,
      useCORS: true,
      letterRendering: true,
      logging: false
    },
    jsPDF: {
      unit: 'mm',
      format: 'a4',
      orientation: 'portrait',
      compress: true
    }
  }

  html2pdf().set(opt).from(wrapper).save().then(() => {
    closeSaveDialog()
  })
}

onBeforeUnmount(() => {
  editor.value?.destroy()
})
</script>

<style>
.editor-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  background-color: #f8f9fa;
}

/* ProseMirror Editor Styles */
:deep(.ProseMirror) {
  min-height: 100%;
  outline: none !important;
  border: none !important;
  cursor: text;
  font-family: Arial, sans-serif;
  font-size: 11pt;
  line-height: 1.5;
}

:deep(.ProseMirror p) {
  margin: 0 0 10pt 0;
  min-height: 1em;
}

:deep(.ProseMirror p:first-child) {
  margin-top: 0;
}

:deep(.ProseMirror h1) {
  font-size: 24pt;
  font-weight: bold;
  margin: 0 0 12pt 0;
  line-height: 1.3;
}

:deep(.ProseMirror h2) {
  font-size: 20pt;
  font-weight: bold;
  margin: 0 0 12pt 0;
  line-height: 1.3;
}

:deep(.ProseMirror h3) {
  font-size: 16pt;
  font-weight: bold;
  margin: 0 0 10pt 0;
  line-height: 1.3;
}

:deep(.ProseMirror h4) {
  font-size: 14pt;
  font-weight: bold;
  margin: 0 0 10pt 0;
  line-height: 1.3;
}

:deep(.ProseMirror h5) {
  font-size: 12pt;
  font-weight: bold;
  margin: 0 0 10pt 0;
  line-height: 1.3;
}

:deep(.ProseMirror h6) {
  font-size: 11pt;
  font-weight: bold;
  margin: 0 0 10pt 0;
  line-height: 1.3;
}

/* List styles - completely reset and rebuild */
:deep(.ProseMirror ul.tiptap-bullet-list),
:deep(.ProseMirror ol.tiptap-ordered-list) {
  margin: 0 0 10pt 0 !important;
  padding: 0 !important;
  padding-left: 30px !important;
  list-style-position: outside !important;
}

:deep(.ProseMirror ul.tiptap-bullet-list) {
  list-style-type: disc !important;
}

:deep(.ProseMirror ol.tiptap-ordered-list) {
  list-style-type: decimal !important;
}

:deep(.ProseMirror li.tiptap-list-item) {
  margin: 0 0 2pt 0 !important;
  padding: 0 !important;
  padding-left: 5px !important;
  display: list-item !important;
}

:deep(.ProseMirror li.tiptap-list-item p) {
  margin: 0 !important;
  padding: 0 !important;
  display: inline !important;
}

:deep(.ProseMirror p.is-editor-empty:first-child::before) {
  color: #adb5bd;
  content: attr(data-placeholder);
  float: left;
  height: 0;
  pointer-events: none;
}

:deep(.ProseMirror:focus) {
  outline: none !important;
  border: none !important;
}

:deep(.ProseMirror-focused) {
  outline: none !important;
  border: none !important;
}

:deep(.ProseMirror *) {
  outline: none !important;
  border: none !important;
}

:deep(.ProseMirror p:focus) {
  outline: none !important;
  border: none !important;
}

:deep(.ProseMirror *:focus) {
  outline: none !important;
  border: none !important;
}
</style>