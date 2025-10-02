<script setup lang="ts">
import { ref } from 'vue';

const count = ref(0);

const increment = () => {
  count.value++;
};

const decrement = () => {
  count.value--;
};
</script>

<template>
  <div>
    <p>compteur : {{ count }}</p>
    <button @click="increment">increment</button>
    <button @click="decrement">decrement</button>
  </div>
</template>

<style scoped>
button {
  margin-right: 10px;
  background-color: black;
  color: white;
  border: none;
  padding: 10px;
}
</style>
