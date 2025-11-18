<script setup lang="ts">
import { ref, onMounted } from 'vue';

interface Post {
  _id?: string;
  _rev?: string;
  post_name: string;
  post_content: string;
  comments?: any[];
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

const title = ref('');
const content = ref('');
const currentDoc = ref<Post | null>(null);

onMounted(async () => {
  try {
    const doc = await props.storage.get(props.documentId);
    currentDoc.value = doc;
    title.value = doc.post_name || '';
    content.value = doc.post_content || '';
  } catch (err) {
    console.error('Erreur lors du chargement du document:', err);
    alert('Impossible de charger le document');
    emit('close');
  }
});

const handleSubmit = (e: Event) => {
  e.preventDefault();

  if (!title.value || !content.value) {
    alert('Veuillez remplir tous les champs');
    return;
  }

  if (!currentDoc.value) {
    alert('Document non chargé');
    return;
  }

  // Mettre à jour le document existant (garder _id et _rev)
  const updatedDoc = {
    ...currentDoc.value,
    post_name: title.value,
    post_content: content.value,
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
          <button type="submit">Modifier</button>
        </div>
      </form>
    </div>
  </div>
</template>

<style scoped></style>
