<script >
export default {
  data() {
    return {
      counter: 0
    }
  },
}
</script>

<template>
  <div>
    {{ counter }}
    <br />
    <button @click="counter++">
      increment
    </button>
  </div>
</template>

<style scoped>
</style>
