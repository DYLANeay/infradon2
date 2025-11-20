<script setup lang="ts">
import { ref } from 'vue';

interface Props {
  storage: any;
  isOnline?: boolean;
}

const props = defineProps<Props>();
const emit = defineEmits<{
  close: [];
  documentAdded: [];
}>();

const bookName = ref('');
const bookDescription = ref('');
const bookCategory = ref('');
const bookAuthor = ref('');
const bookLikes = ref(0);

const handleSubmit = (e: Event) => {
  e.preventDefault();

  if (!bookName.value || !bookDescription.value) {
    alert('Veuillez remplir tous les champs obligatoires');
    return;
  }

  const newDoc = {
    book_name: bookName.value,
    book_description: bookDescription.value,
    book_category: bookCategory.value,
    book_author: bookAuthor.value,
    book_likes: bookLikes.value,
    review_comments: [],
    attributes: {
      creation_date: new Date().toISOString(),
    },
  };

  props.storage
    .post(newDoc)
    .then((response: any) => {
      console.log('Document ajouté avec succès : ', response);
      bookName.value = '';
      bookDescription.value = '';
      bookCategory.value = '';
      bookAuthor.value = '';
      bookLikes.value = 0;
      emit('documentAdded');
      emit('close');
    })
    .catch((err: any) => {
      console.error("Erreur lors de l'ajout du document : ", err);
      alert("Erreur lors de l'ajout du document");
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
        <h2>Ajouter un document</h2>
        <button @click="handleClose">&times;</button>
      </div>
      <p v-if="props.isOnline === false">
        Vous êtes hors ligne — le document sera stocké localement et synchronisé lorsque la
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
          <label for="doc-likes">Nombre de likes:</label>
          <input
            type="number"
            id="doc-likes"
            v-model="bookLikes"
            placeholder="Entrez le nombre de likes"
          />
        </div>
        <div>
          <button type="button" @click="handleClose">Annuler</button>
          <button type="submit">Ajouter</button>
        </div>
      </form>
    </div>
  </div>
</template>

<style scoped></style>
