<script setup>
import { ref } from 'vue'
import TextEditor from './components/TextEditor.vue'
import PDFSigner from './pages/PDFSigner.vue'

const mode = ref('editor') // 'editor' or 'pdf'
</script>

<template>
  <div class="app-container">
    <!-- Mode Switcher -->
    <div class="mode-switcher">
      <button
        :class="{ active: mode === 'editor' }"
        @click="mode = 'editor'"
      >
        Text Editor
      </button>
      <button
        :class="{ active: mode === 'pdf' }"
        @click="mode = 'pdf'"
      >
        PDF Signer
      </button>
    </div>

    <!-- Content -->
    <TextEditor v-if="mode === 'editor'" />
    <PDFSigner v-else />
  </div>
</template>

<style>
.app-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
}

.mode-switcher {
  display: flex;
  gap: 0;
  background: #202124;
  padding: 5px;
}

.mode-switcher button {
  padding: 10px 20px;
  border: none;
  background: transparent;
  color: #e8eaed;
  cursor: pointer;
  font-size: 14px;
  border-radius: 4px;
  transition: background 0.2s;
}

.mode-switcher button:hover {
  background: rgba(255, 255, 255, 0.1);
}

.mode-switcher button.active {
  background: #4285f4;
  color: white;
}
</style>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html, body, #app {
  height: 100vh;
  width: 100vw;
  overflow: hidden;
}
</style>