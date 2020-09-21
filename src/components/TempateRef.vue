<template>
  <div ref="root1">This is a root element 1</div>
  <div ref="root2">This is a root element 2</div>
</template>

<script>
import { onMounted, ref } from 'vue'
export default {
  setup() {
    const root1 = ref(null)
    const root2 = ref(null)
    onMounted(() => {
      console.info(root1.value)
      console.info(root2.value)
    })
    return {
      root1,
      root2
    }
  }
}
</script>

<style>
</style>