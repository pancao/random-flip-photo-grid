<template>
  <div id="app">
    <ColorGrid />
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue'
import ColorGrid from './components/ColorGrid.vue' // 修改为你的组件路径

export default defineComponent({
  components: {
    ColorGrid
  },
});
</script>

<style>
#app {
  height: 100vh;
  width: 100vw;
  margin: 0;
  padding: 0;
}
</style>
