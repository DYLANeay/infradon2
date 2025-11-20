<script setup lang="ts">
import { ref, onMounted } from 'vue';

interface Book {
  _id?: string;
  _rev?: string;
  book_name: string;
  book_description: string;
  review_title?: string;
  book_category?: string;
  book_author?: string;
  review_comments: { title: string; author: string }[];
  attributes?: {
    creation_date: any;
  };
}

interface Props {
  storage: any;
  documentId: string;
  isOnline?: boolean;
}

const props = defineProps<Props>();
const emit = defineEmits<{
  close: [];
  documentModified: [];
}>();

const bookName = ref('');
const bookDescription = ref('');
const bookCategory = ref('');
const bookAuthor = ref('');
const currentDoc = ref<Book | null>(null);

onMounted(async () => {
  try {
    const doc = await props.storage.get(props.documentId);
    currentDoc.value = doc;
    bookName.value = doc.book_name || '';
    bookDescription.value = doc.book_description || '';
    bookCategory.value = doc.book_category || '';
    bookAuthor.value = doc.book_author || '';
  } catch (err) {
    console.error('Erreur lors du chargement du document:', err);
    alert('Impossible de charger le document');
    emit('close');
  }
});

const handleSubmit = (e: Event) => {
  e.preventDefault();

  if (!bookName.value || !bookDescription.value) {
    alert('Veuillez remplir tous les champs obligatoires');
    return;
  }

  if (!currentDoc.value) {
    alert('Document non chargé');
    return;
  }

  // Mettre à jour le document existant (garder _id et _rev)
  const updatedDoc = {
    ...currentDoc.value,
    book_name: bookName.value,
    book_description: bookDescription.value,
    book_category: bookCategory.value,
    book_author: bookAuthor.value,
    attributes: {
      ...currentDoc.value.attributes,
      modification_date: new Date().toISOString(),
    },
  };

  props.storage
    .put(updatedDoc)
    .then((response: any) => {
      console.log('Document modifié avec succès : ', response);
      emit('documentModified');
      emit('close');
    })
    .catch((err: any) => {
      console.error('Erreur lors de la modification du document : ', err);
      alert('Erreur lors de la modification du document');
    });
};

const handleClose = () => {
  emit('close');
};
</script>

<template>
  <div @click.self="handleClose">
    <div>
      <div>
        <h2>Modifier le document</h2>
        <button @click="handleClose">&times;</button>
      </div>
      <p v-if="props.isOnline === false">
        Vous êtes hors ligne — la modification sera appliquée localement et synchronisée lorsque la
        connexion reviendra.
      </p>
      <form @submit="handleSubmit">
        <div>
          <label for="doc-name">Nom du livre:</label>
          <input
            type="text"
            id="doc-name"
            v-model="bookName"
            placeholder="Entrez le nom du livre"
            required
          />
        </div>
        <div>
          <label for="doc-description">Description:</label>
          <textarea
            id="doc-description"
            v-model="bookDescription"
            placeholder="Entrez la description"
            rows="6"
            required
          ></textarea>
        </div>
        <div>
          <label for="doc-category">Catégorie:</label>
          <input
            type="text"
            id="doc-category"
            v-model="bookCategory"
            placeholder="Ex: philo, psycho, roman, poésie..."
          />
        </div>
        <div>
          <label for="doc-author">Auteur:</label>
          <input
            type="text"
            id="doc-author"
            v-model="bookAuthor"
            placeholder="Entrez le nom de l'auteur"
          />
        </div>
        <div>
          <button type="button" @click="handleClose">Annuler</button>
          <button type="submit">Modifier</button>
        </div>
      </form>
    </div>
  </div>
</template>

<style scoped></style>
