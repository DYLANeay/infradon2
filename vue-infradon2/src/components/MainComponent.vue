<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref } from 'vue';
import PouchDB from 'pouchdb';
import PouchDBFind from 'pouchdb-find';
import FormDoc from './FormDocAdd.vue';
import FormDocModify from './FormDocModify.vue';

PouchDB.plugin(PouchDBFind);

declare interface Book {
  _id?: string;
  book_name: string;
  book_description: string;
  review_title?: string;
  book_category?: string;
  book_author?: string;
  book_likes: number;
  review_comments: { content: string; author: string }[];
  attributes: {
    creation_date: any;
  };
}

const storage = ref(); // considered as main db i guess
const localDB = ref();
const remoteDB = ref();
const isOnline = ref(true);
const simulatedOffline = ref(false);
// data stockées
const booksData = ref<Book[]>([]);
const showPopupRef = ref(false);
const showModifyPopupRef = ref(false);
const documentToModify = ref<string | null>(null);
const syncStatus = ref('');
const searchTerm = ref('');
const searchResults = ref<Book[]>([]);
const isSearching = ref(false);

// Factory pour générer des documents de test
const generateTestDocuments = async (count: number = 50) => {
  console.log(`Génération de ${count} documents de test...`);
  const categories = ['philo', 'psycho', 'roman', 'poésie', 'récit', 'nouvelle'];
  const authors = ['Dostoïevski', 'Camus', 'Jung', 'Kourkov', 'Voltaire','Hugo'];
  const docs = [];
  for (let i = 0; i < count; i++) {
    const category = categories[Math.floor(Math.random() * categories.length)];
    const author = authors[Math.floor(Math.random() * authors.length)];

    docs.push({
      book_name: `Document ${i + 1} - ${category}`,
      book_description: `Contenu du document ${i + 1} sur ${category}. Lorem ipsum dolor sit amet.`,
      book_category: category,
      book_author: author,
      review_comments: [
        { title: "Review Title 1", author: "Reviewer 1" },
        { title: "Review Title 2", author: "Reviewer 2" }
      ],
      book_likes: Math.floor(Math.random() * 100),
      attributes: {
        //date random
        creation_date: new Date(
          Date.now() - Math.random() * 365 * 24 * 60 * 60 * 1000,
        ).toISOString(),
      },
    });
  }

  try {
    await storage.value.bulkDocs(docs);
    console.log(`${count} documents générés avec succès`);
    fetchData();
  } catch (err) {
    console.error('Erreur lors de la génération des documents:', err);
  }
};

const createIndex = async () => {
  try {
    await storage.value.createIndex({
      index: {
        fields: ['book_category',]
      },
    });
    console.log('Index créé sur le champ book_category et book_author');
  } catch (err) {
    console.error("Erreur lors de la création de l'index:", err);
  }
};

// Recherche par catégorie
const searchByCategory = async () => {
  const term = searchTerm.value.trim().toLowerCase();

  if (!term) {
    searchResults.value = [];
    isSearching.value = false;
    return;
  }

  isSearching.value = true;
  try {
    console.log(`Recherche pour la catégorie: "${term}"`);

    // Recherche exacte uniquement
    const result = await storage.value.find({
      selector: {
        book_category: { $eq: term },
      },
    });

    searchResults.value = result.docs;
    console.log(`Recherche terminée: ${result.docs.length} résultats pour "${searchTerm.value}"`);
  } catch (err) {
    console.error('Erreur lors de la recherche:', err);
    // Fallback: recherche js dans tous les documents
    try {
      const allDocs = await storage.value.allDocs({ include_docs: true });
      searchResults.value = allDocs.rows
        .map((row: any) => row.doc)
        .filter((doc: any) => doc.book_category && doc.book_category.toLowerCase() === term);
      console.log(`Recherche fallback: ${searchResults.value.length} résultats`);
    } catch (fallbackErr) {
      console.error('Erreur lors de la recherche fallback:', fallbackErr);
      searchResults.value = [];
    }
  }
  isSearching.value = false;
};

// Initialisation de la base de données
const initDatabase = async () => {
  console.log('=> Initialisation des bases de données');

  localDB.value = new PouchDB('infradon2_local_db');
  console.log('Base de données locale créée : infradon2_local_db');

  remoteDB.value = new PouchDB('http://admin:1756@localhost:5984/infradon2_db');
  console.log('Connecté à la base de données distante : infradon2_db');

  storage.value = localDB.value;

  await createIndex();

  syncFromRemote();
};

const syncFromRemote = () => {
  console.log('Synchronisation depuis la base distante vers la base locale');
  syncStatus.value = 'Téléchargement des données distantes...';

  localDB.value.replicate
    .from(remoteDB.value)
    .on('complete', (info: any) => {
      console.log('Synchronisation depuis la base distante terminée', info);
      syncStatus.value = 'Données distantes synchronisées';
      fetchData();
      setTimeout(() => {
        syncStatus.value = '';
      }, 3000);
    })
    .on('error', (err: any) => {
      console.error('Erreur lors de la synchronisation depuis la base distante :', err);
      syncStatus.value = 'Erreur de synchronisation';
    });
};

const syncToRemote = () => {
  console.log('Synchronisation depuis la base locale vers la base distante');
  syncStatus.value = 'Envoi des données locales...';

  localDB.value.replicate
    .to(remoteDB.value)
    .on('complete', (info: any) => {
      console.log('Synchronisation vers la base distante terminée', info);
      syncStatus.value = 'Données locales envoyées';
      setTimeout(() => {
        syncStatus.value = '';
      }, 3000);
    })
    .on('error', (err: any) => {
      console.error('Erreur lors de la synchronisation vers la base distante :', err);
      syncStatus.value = "Erreur d'envoi";
    });
};

// Synchronisation bidirectionnelle complète
const syncDatabase = () => {
  console.log('Synchronisation bidirectionnelle entre local et distant');
  syncStatus.value = 'Synchronisation en cours...';

  localDB.value
    .sync(remoteDB.value)
    .on('complete', (info: any) => {
      console.log('Synchronisation bidirectionnelle terminée', info);
      syncStatus.value = 'Synchronisation réussie';
      fetchData();
      setTimeout(() => {
        syncStatus.value = '';
      }, 3000);
    })
    .on('error', (err: any) => {
      console.error('Erreur lors de la synchronisation :', err);
      syncStatus.value = 'Erreur de synchronisation';
    })
    .on('change', (info: any) => {
      console.log('Changements détectés:', info);
    });
};

// Récupération des données
const fetchData = () => {
  console.log('=> Récupération des données');
  storage.value
    .allDocs({ include_docs: true, descending: true })
    .then((result: any) => {
      console.log(result);
      booksData.value = result.rows.map((row: any) => row.doc) as Book[];
      console.log('Données récupérées : ', booksData.value);
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

const showPopup = () => {
  showPopupRef.value = true;
};

const toggleSimulatedOffline = () => {
  simulatedOffline.value = !simulatedOffline.value;
  if (simulatedOffline.value) {
    // force offline state
    isOnline.value = false;
    console.log('Simulation: hors ligne activée');
  } else {
    // restore online state and trigger sync
    isOnline.value = true;
    console.log('Simulation: hors ligne désactivée');
    if (isOnline.value) {
      syncDatabase();
    }
  }
};

const closePopup = () => {
  showPopupRef.value = false;
  fetchData();
};

const closeModifyPopup = () => {
  showModifyPopupRef.value = false;
  documentToModify.value = null;
  fetchData();
};

const handleDocumentAdded = () => {
  fetchData();
  searchResults.value = [];

  if (isOnline.value) {
    syncToRemote();
  }
};

const handleDocumentModified = () => {
  fetchData();

  if (isOnline.value) {
    syncToRemote();
  }
};

onMounted(async () => {
  console.log('=> Composant initialisé');
  await initDatabase();
  fetchData();
  const handleOnline = () => {
    console.log('Network: online');
    if (simulatedOffline.value) {
      console.log('Network online ignored because simulated offline is active');
      return;
    }
    isOnline.value = true;
    // when back online, run a full sync
    syncDatabase();
  };

  const handleOffline = () => {
    console.log('Network: offline');
    if (simulatedOffline.value) {
      console.log('Network offline event ignored because simulated offline is active');
      return;
    }
    isOnline.value = false;
  };

  window.addEventListener('online', handleOnline);
  window.addEventListener('offline', handleOffline);

  // remove listeners on unmount
  onBeforeUnmount(() => {
    window.removeEventListener('online', handleOnline);
    window.removeEventListener('offline', handleOffline);
  });
});
</script>

<template>
  <h1>Fetch Data</h1>

  <div class="action-buttons">
    <button @click="showPopup">Ajouter un document</button>
    <button @click="syncDatabase">Synchroniser</button>
    <button @click="syncFromRemote">Télécharger</button>
    <button @click="syncToRemote">Envoyer</button>
    <button @click="toggleSimulatedOffline">Basculer mode hors ligne simulé</button>
    <button @click="generateTestDocuments(50)">Générer 50 documents test</button>
  </div>

  <div>
    <input
      type="text"
      v-model="searchTerm"
      placeholder="Rechercher par catégorie"
      @input="searchByCategory"
    />
    <button
      v-if="searchTerm"
      @click="
        searchTerm = '';
        searchResults = [];
      "
    >
      Effacer recherche
    </button>
    <p v-if="isSearching">Recherche en cours...</p>
    <p v-if="searchResults.length > 0">{{ searchResults.length }} résultat trouvés</p>
    <p v-if="searchTerm && !isSearching && searchResults.length === 0">
      Aucun résultat trouvé pour "{{ searchTerm }}"
    </p>
  </div>

  <p v-if="syncStatus">{{ syncStatus }}</p>
  <p>
    Statut réseau:
    {{ simulatedOffline ? 'Hors ligne (simulé)' : isOnline ? 'En ligne' : 'Hors ligne' }}
  </p>

  <!-- Formulaire d'ajout (autre component) -->
  <FormDoc
    v-if="showPopupRef"
    :storage="storage"
    :is-online="isOnline"
    @close="closePopup"
    @document-added="handleDocumentAdded"
  />

  <!-- Formulaire de modification (autre component))-->
  <FormDocModify
    v-if="showModifyPopupRef && documentToModify"
    :storage="storage"
    :document-id="documentToModify"
    :is-online="isOnline"
    @close="closeModifyPopup"
    @document-modified="handleDocumentModified"
  />

  <div v-if="searchResults.length > 0">
    <h3>Résultats de recherche :</h3>
    <article v-for="book in searchResults" v-bind:key="book._id">
      <h2>{{ book.book_name }}</h2>
      <p><strong>Catégorie:</strong> {{ book.book_category }}</p>
      <p><strong>Auteur:</strong> {{ book.book_author }}</p>
      <h4>{{ book.book_description }}</h4>
      <!-- on filtre pour ne pas afficher les boutons sur les index -->
      <div v-if="!book._id?.startsWith('_design/')">
            <h3>this book has {{ book.book_likes }} likes</h3>
        <button @click="updateDocument(book._id as any)">Modifier le document</button>
        <button @click="deleteDocument(book._id as any)">Supprimer le document</button>
      </div>
    </article>
  </div>

  <div v-else>
    <h3>Tous les documents :</h3>
    <article v-for="book in booksData" v-bind:key="book._id">
      <h2>{{ book.book_name }}</h2>
      <p v-if="book.book_category"><strong>Catégorie:</strong> {{ book.book_category }}</p>
      <p v-if="book.book_author"><strong>Auteur:</strong> {{ book.book_author }}</p>
      <h4>{{ book.book_description }}</h4>
      <p v-for="(review_comments, index) in book.review_comments" v-bind:key="index">
        {{ review_comments.author }} said : {{ review_comments.content }}
      </p>
      <div v-if="!book._id?.startsWith('_design/')">
        <h3>this book has {{ book.book_likes }} likes</h3>
        <button @click="updateDocument(book._id as any)">Modifier le document</button>
        <button @click="deleteDocument(book._id as any)">Supprimer le document</button>
      </div>
    </article>
  </div>
</template>
