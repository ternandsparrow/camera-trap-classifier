<template>
  <div id="app">
    <ctc-classifier />
  </div>
</template>

<script>
import Classifier from '@/components/Classifier'

export default {
  name: 'App',
  components: {
    'ctc-classifier': Classifier,
  },
}
</script>

<style lang="scss" scoped>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
}
</style>
