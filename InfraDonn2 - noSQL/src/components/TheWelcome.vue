<script setup lang="ts">
import { ref } from 'vue'
import PouchDB from 'pouchdb';

const counter = ref(0);

const increment = () => {
  counter.value++
}

function mounted() {
    this.initDatabase()
}

function initDatabase() {
  const db = new PouchDB('http://admin:admin@localhost:5984/test_infradonn2')
}

</script>

<template>
  <h1>Welcome</h1>
  <p>Counter: {{ counter }}</p>
  <button @click="increment">Increment</button>
</template>
