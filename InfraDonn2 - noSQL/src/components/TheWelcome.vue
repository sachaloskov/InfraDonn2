<script setup lang="ts">
import { onMounted, ref } from 'vue';
import PouchDB from 'pouchdb'

declare interface Post {
  post_name: string;
  post_content: string;
  attributes: {
    creation_date: any;
  };
}

// Référence à la base de données
const storage = ref()
// Données stockées
const postsData = ref<Post[]>([])

// Initialisation de la base de données
const initDatabase = () => {
  console.log('=> Connexion à la base de données');
  const db = new PouchDB('http://admin:admin@localhost:5984/test_infradonn2')
  if (db) {
    console.log("Connecté à la collection : " + db?.name)
    storage.value = db
  } else {
    console.warn('Echec lors de la connexion à la base de données')
  }
}

// Récupération des données
const fetchData = async () => {
  if (!storage.value) {
    console.warn('Base de données non initialisée');
    return;
  }
 
  try {
    console.log('=> Récupération des données...');
    const result = await storage.value.allDocs({ include_docs: true });
 
    // On remplit postsData avec les documents récupérés
    postsData.value = result.rows
      .filter((row: any) => !!row.doc)
      .map((row: any) => row.doc as Post);
 
    console.log('Données récupérées :', postsData.value);
  } catch (err) {
    console.error('Erreur lors du fetch des données :', err);
  }
}
 
onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase()
  fetchData()
});

</script>

<template>
  <h1>Fetch Data</h1>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <h2>{{ post.post_name }}</h2>
    <p>{{ post.post_content }}</p>
  </article>
</template>