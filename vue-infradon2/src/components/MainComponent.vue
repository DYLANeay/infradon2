<script setup lang="ts">
import { onMounted, ref } from 'vue';
import PouchDB from 'pouchdb';
import FormDoc from './FormDocAdd.vue';
import FormDocModify from './FormDocModify.vue';

declare interface Post {
  _id?: string;
  post_name: string;
  post_content: string;
  title?: string;
  comments: { title: string; author: string }[];
  attributes: {
    creation_date: any;
  };
}

// Référence à la base de données
const storage = ref();
// Données stockées
const postsData = ref<Post[]>([]);
const showPopupRef = ref(false);
const showModifyPopupRef = ref(false);
const documentToModify = ref<string | null>(null);

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
      console.log(result);
      postsData.value = result.rows.map((row: any) => row.doc) as Post[];
      console.log('Données récupérées : ', postsData.value);
    })
    .catch((err: any) => {
      console.error('Erreur lors de la récupération des données : ', err);
    });
};

//modifier un document
const updateDocument = (id: string) => {
  documentToModify.value = id;
  showModifyPopupRef.value = true;
};

//delete a document
const deleteDocument = (id: string) => {
  storage.value
    .get(id)
    .then((doc: any) => {
      return storage.value.remove(doc);
    })
    .then((response: any) => {
      console.log('Document supprimé avec succès : ', response);
      fetchData(); //refresh data
    })
    .catch((err: any) => {
      console.error('Erreur lors de la suppression du document : ', err);
    });
};

//replicate database
const replicateDatabase = () => {};

const showPopup = () => {
  showPopupRef.value = true;
};

const closePopup = () => {
  showPopupRef.value = false;
};

const closeModifyPopup = () => {
  showModifyPopupRef.value = false;
  documentToModify.value = null;
};

const handleDocumentAdded = () => {
  fetchData();
};

const handleDocumentModified = () => {
  fetchData();
};

onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase();
  fetchData();
});
</script>

<template>
  <h1>Fetch Data</h1>
  <button @click="showPopup">Ajouter un document</button>

  <!-- Formulaire d'ajout -->
  <FormDoc
    v-if="showPopupRef"
    :storage="storage"
    @close="closePopup"
    @document-added="handleDocumentAdded"
  />

  <!-- Formulaire de modification -->
  <FormDocModify
    v-if="showModifyPopupRef && documentToModify"
    :storage="storage"
    :document-id="documentToModify"
    @close="closeModifyPopup"
    @document-modified="handleDocumentModified"
  />

  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <h2>{{ post.post_name }}</h2>
    <h3>{{ post.post_content }}</h3>
    <p v-for="(comment, index) in post.comments" v-bind:key="index">
      {{ comment.author }} said : {{ comment.title }}
    </p>
    <button @click="updateDocument(post._id as any)">Modifier le document</button>
    <button @click="deleteDocument(post._id as any)">Supprimer le document</button>
    <!--     <p v-for="comment in post.comments" v-bind:key="comment.id">{{ comment.content }}</p>
 -->
  </article>
</template>
