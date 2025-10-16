<script setup lang="ts">
import { onMounted, ref } from 'vue';
import PouchDB from 'pouchdb';

declare interface Post {
  post_name: string;
  post_content: string;
  attributes: {
    creation_date: any;
  };
}

// Référence à la base de données
const storage = ref();
// Données stockées
const postsData = ref<Post[]>([]);

// Initialisation de la base de données
const initDatabase = () => {
  console.log('=> Connexion à la base de données');

  const db = new PouchDB('http://admin:1756@localhost:5984/infradon2_db');
  if (db) {
    console.log('Connecté à la collection : ' + db?.name);
    storage.value = db;
  } else {
    console.warn('Echec lors de la connexion à la base de données');
  }
};

// Récupération des données
const fetchData = () => {
  console.log('=> Récupération des données');
  storage.value
    .allDocs({ include_docs: true, descending: true })
    .then((result: any) => {
      postsData.value = result.rows.map((row: any) => row.doc) as Post[];
      console.log('Données récupérées : ', postsData.value);
      console.log(postsData.value[0]?.post_name);
    })
    .catch((err: any) => {
      console.error('Erreur lors de la récupération des données : ', err);
    });
};

// Ajouter des documents
const addDocument = () => {
  const newDoc = {
    post_name: 'Nouveau Post',
    post_content: 'Contenu du nouveau post',
    comments: [
      { title: 'Great post!', author: 'Alice', id: 1 },
      { title: 'Thanks for sharing.', author: 'Bob', id: 2 },
    ],
    attributes: {
      creation_date: new Date().toISOString(),
    },
  };

  storage.value
    .post(newDoc)
    .then((response: any) => {
      console.log('Document ajouté avec succès : ', response);
      fetchData(); // Rafraîchir les données après l'ajout
    })
    .catch((err: any) => {
      console.error("Erreur lors de l'ajout du document : ", err);
    });
};

//modifier un document
const updateDocument = () => {};

onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase();
  fetchData();
});
</script>

<template>
  <h1>Fetch Data</h1>
  <button @click="addDocument">Ajouter un document</button>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <h2>{{ post.post_name }}</h2>
    <h3>{{ post.post_content }}</h3>
    <p v-for="comment in post.comments" v-bind:key="comment.id">
      {{ comment.author }} said {{ comment.title }}
    </p>
    <!--     <p v-for="comment in post.comments" v-bind:key="comment.id">{{ comment.content }}</p>
 -->
  </article>
</template>
