<script setup lang="ts">
import { ref } from 'vue';

//DATA => Model
//mise à jour automatique du DOM lorsque la valeur de count change
const count = ref(0);

//METHODS => Controller
const increment = () => {
  count.value++;
};

const decrement = () => {
  count.value--;
};
</script>

<!-- TEMPLATE => View -->
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
