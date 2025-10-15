<script setup>
import PDFSigner from './pages/PDFSigner.vue'
</script>

<template>
  <div class="app-container">
    <PDFSigner />
  </div>
</template>

<style>
.app-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
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