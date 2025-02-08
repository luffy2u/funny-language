<script setup lang="ts">
import LanguageSelector from './components/LanguageSelector.vue'
</script>

<template>
  <LanguageSelector />
</template>

<style>
/* 重置所有元素的默认样式 */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

/* 移除所有可能的滚动条和溢出 */
html, body {
  width: 100%;
  height: 100%;
  margin: 0;
  padding: 0;
  overflow: hidden;
}

/* 重置根元素样式 */
:root {
  overflow: hidden;
}

/* 确保 app 容器完全铺满 */
#app {
  width: 100%;
  height: 100%;
  margin: 0;
  padding: 0;
  overflow: hidden;
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
}
</style>
