<script setup lang="ts">
import {ref, watch} from 'vue'
import SpriteSheetAnim from "./components/SpriteSheetAnim.vue";


const maxNum = 500000
const n = ref(1000)
// Clamp to a reasonable range
watch(n, (v, _o, r) => {
  const vv = Math.max(1, Math.min(maxNum, Math.floor(Number(v) || 1)))
  if (vv !== v) n.value = vv
})
</script>

<template>
  <div class="app">

    <SpriteSheetAnim :numberOfCharacters="n"/>
    <div class="controls">
      <label>
        numberOfCharacters:
        <input type="range" min="1" :max="maxNum" step="1" v-model.number="n"/>
      </label>
      <input class="num" type="number" min="1" :max="maxNum" step="1" v-model.number="n"/>
      <span class="val">{{ n }}</span>
    </div>
  </div>
</template>

<style scoped>
.app {
  padding: 12px;
  color: #ddd;
}

.controls {
  position: fixed;
  bottom: 12px;
  left: 12px;
  display: flex;
  gap: 8px;
  align-items: center;
  margin-bottom: 8px;
  z-index: 12
}

.controls label {
  display: flex;
  gap: 8px;
  align-items: center;
}

.num {
  width: 100px;
}

.val {
  min-width: 48px;
  display: inline-block;
  text-align: right;
}
</style>


