<template>
  <div class="toolbar">
    <select @change="$emit('set-heading', $event)" class="heading-select">
      <option value="paragraph">Normal text</option>
      <option value="1">Heading 1</option>
      <option value="2">Heading 2</option>
      <option value="3">Heading 3</option>
      <option value="4">Heading 4</option>
      <option value="5">Heading 5</option>
      <option value="6">Heading 6</option>
    </select>
    
    <div class="separator"></div>
    
    <button
      @click="$emit('toggle-bold')"
      :class="{ 'is-active': isBold }"
    >
      <strong>B</strong>
    </button>
    <button
      @click="$emit('toggle-italic')"
      :class="{ 'is-active': isItalic }"
    >
      <em>I</em>
    </button>
    <button
      @click="$emit('toggle-underline')"
      :class="{ 'is-active': isUnderline }"
    >
      <u>U</u>
    </button>
    <button
      @click="$emit('toggle-highlight')"
      :class="{ 'is-active': isHighlight }"
    >
      Highlight
    </button>
    
    <div class="separator"></div>
    
    <button
      @click="$emit('toggle-bullet-list')"
      :class="{ 'is-active': isBulletList }"
      title="Bullet List"
    >
      • List
    </button>
    <button
      @click="$emit('toggle-ordered-list')"
      :class="{ 'is-active': isOrderedList }"
      title="Numbered List"
    >
      1. List
    </button>
    
    <div class="separator"></div>
    
    <select :value="alignment" @change="$emit('set-alignment', $event)">
      <option value="left">Align Left</option>
      <option value="center">Align Center</option>
      <option value="right">Align Right</option>
      <option value="justify">Justify</option>
    </select>
  </div>
</template>

<script setup>
defineProps({
  isBold: Boolean,
  isItalic: Boolean,
  isUnderline: Boolean,
  isHighlight: Boolean,
  isBulletList: Boolean,
  isOrderedList: Boolean,
  alignment: String
})

defineEmits([
  'set-heading',
  'toggle-bold',
  'toggle-italic',
  'toggle-underline',
  'toggle-highlight',
  'toggle-bullet-list',
  'toggle-ordered-list',
  'set-alignment'
])
</script>

<style scoped>
.toolbar {
  display: flex;
  gap: 8px;
  padding: 8px;
  background-color: #fff;
  border-bottom: 1px solid #e0e0e0;
}

.toolbar button {
  padding: 8px 14px;
  border: 1px solid #e0e0e0;
  background-color: #fff;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
}

.toolbar button:hover {
  background-color: #f1f3f4;
}

.toolbar button.is-active {
  background-color: #e8f0fe;
  color: #1a73e8;
}

.toolbar select {
  padding: 8px 10px;
  border: 1px solid #e0e0e0;
  border-radius: 4px;
  background-color: #fff;
  cursor: pointer;
  font-size: 15px;
}

.toolbar .heading-select {
  min-width: 150px;
}

.toolbar .separator {
  width: 1px;
  height: 24px;
  background-color: #e0e0e0;
  margin: 0 4px;
}
</style>
