<script setup lang="ts">
import { onMounted, ref } from 'vue';
import PouchDB from 'pouchdb';

//DATA => Model
//mise à jour automatique du DOM lorsque la valeur de count change

const storage = ref();
const postsData = ref([]);

onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase();
});

const initDatabase = () => {
  console.log('=> Connexion à la base de données');
  const db = new PouchDB('http://admin:1756@localhost:5984/infradon2_db');
  if (db) {
    console.log('Connected to collection : ' + db?.name);
    storage.value = db;
  } else {
    console.warn('Something went wrong');
  }
};

//not so good
/* const fetchData = async () => {
  if (storage.value) {
    try {
      const result = await storage.value.allDocs({ include_docs: true });
      postsData.value = result.rows.map((row: any) => row.doc);
      console.log('Données récupérées :', postsData.value);
    } catch (error) {
      console.error('Erreur lors de la récupération des données :', error);
    }
  }
}; */

//METHODS => Controller
</script>

<!-- TEMPLATE => View -->
<template>
  <div></div>
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
