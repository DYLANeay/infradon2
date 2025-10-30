<script setup lang="ts">
import { onMounted, ref } from 'vue';
import PouchDB from 'pouchdb';

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
const updateDocument = (id: string) => {
  storage.value
    .get(id)
    .then((doc: any) => {
      doc.post_name = 'Post modifié';
      return storage.value.put(doc);
    })
    .then((response: any) => {
      console.log('Document mis à jour avec succès : ', response);
      fetchData(); //refresh data
    })
    .catch((err: any) => {
      console.error('Erreur lors de la mise à jour du document : ', err);
    });
};

const deleteDocument = (id: string) => {
  storage.value
    .get(id)
    .then((doc: any) => {
      return storage.value.remove(doc);
      fetchData(); //refresh data
    })
    .then((response: any) => {
      console.log('Document supprimé avec succès : ', response);
      fetchData(); //refresh data
    })
    .catch((err: any) => {
      console.error('Erreur lors de la suppression du document : ', err);
    });
};

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
      {{ comment.author }} said : {{ comment.title }}
    </p>
    <button @click="updateDocument(post._id as any)">Modifier le document</button>
    <button @click="deleteDocument(post._id as any)">Supprimer le document</button>
    <!--     <p v-for="comment in post.comments" v-bind:key="comment.id">{{ comment.content }}</p>
 -->
  </article>
</template>
