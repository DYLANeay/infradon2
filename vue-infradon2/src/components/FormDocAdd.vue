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

const title = ref('');
const content = ref('');

const handleSubmit = (e: Event) => {
  e.preventDefault();

  if (!title.value || !content.value) {
    alert('Veuillez remplir tous les champs');
    return;
  }

  const newDoc = {
    post_name: title.value,
    post_content: content.value,
    comments: [],
    attributes: {
      creation_date: new Date().toISOString(),
    },
  };

  props.storage
    .post(newDoc)
    .then((response: any) => {
      console.log('Document ajouté avec succès : ', response);
      title.value = '';
      content.value = '';
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
          <label for="doc-title">Titre:</label>
          <input
            type="text"
            id="doc-title"
            v-model="title"
            placeholder="Entrez le titre"
            required
          />
        </div>
        <div>
          <label for="doc-content">Contenu:</label>
          <textarea
            id="doc-content"
            v-model="content"
            placeholder="Entrez le contenu"
            rows="6"
            required
          ></textarea>
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
