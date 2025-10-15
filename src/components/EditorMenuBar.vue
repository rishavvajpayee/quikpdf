<template>
  <div class="menu-bar">
    <div class="menu-item">
      <button class="menu-button">File</button>
      <div class="dropdown-content">
        <button @click="$emit('new-document')">New</button>
        <button @click="$emit('save')">Save</button>
      </div>
    </div>
    <div class="menu-item">
      <button class="menu-button">Edit</button>
      <div class="dropdown-content">
        <button @click="$emit('undo')">Undo</button>
        <button @click="$emit('redo')">Redo</button>
      </div>
    </div>
    <div class="zoom-control">
      <button @click="$emit('decrease-zoom')">-</button>
      <span>{{ Math.round(zoom * 100) }}%</span>
      <button @click="$emit('increase-zoom')">+</button>
    </div>
  </div>
</template>

<script setup>
defineProps({
  zoom: {
    type: Number,
    required: true
  }
})

defineEmits(['new-document', 'save', 'undo', 'redo', 'decrease-zoom', 'increase-zoom'])
</script>

<style scoped>
.menu-bar {
  display: flex;
  align-items: center;
  background-color: #fff;
  border-bottom: 1px solid #e0e0e0;
  padding: 4px 8px;
}

.zoom-control {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-left: auto;
  padding: 0 16px;
}

.zoom-control button {
  padding: 6px 12px;
  border: 1px solid #e0e0e0;
  background: #fff;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
}

.zoom-control span {
  font-size: 16px;
}

.zoom-control button:hover {
  background-color: #f1f3f4;
}

.menu-item {
  position: relative;
  display: inline-block;
}

.menu-button {
  background: none;
  border: none;
  padding: 10px 16px;
  cursor: pointer;
  font-size: 16px;
}

.menu-button:hover {
  background-color: #f1f3f4;
  border-radius: 4px;
}

.dropdown-content {
  display: none;
  position: absolute;
  background-color: #fff;
  min-width: 160px;
  box-shadow: 0 2px 5px rgba(0,0,0,0.2);
  z-index: 100;
  border-radius: 4px;
}

.menu-item:hover .dropdown-content {
  display: block;
}

.dropdown-content button {
  width: 100%;
  padding: 10px 18px;
  text-align: left;
  border: none;
  background: none;
  cursor: pointer;
  font-size: 15px;
}

.dropdown-content button:hover {
  background-color: #f1f3f4;
}
</style>
