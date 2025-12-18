<script setup lang="ts">
import { ref, onMounted } from "vue";

interface Book {
	_id?: string;
	_rev?: string;
	book_name: string;
	book_description: string;
	review_title?: string;
	book_category?: string;
	book_author?: string;
	book_likes: number;
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

const bookName = ref("");
const bookDescription = ref("");
const bookCategory = ref("");
const bookAuthor = ref("");
const currentDoc = ref<Book | null>(null);
const bookLikes = ref(0);
const selectedFile = ref<File | null>(null);
const imagePreview = ref<string | null>(null);
const hasExistingImage = ref(false);
const removeExistingImage = ref(false);

const handleFileSelect = (event: Event) => {
	const target = event.target as HTMLInputElement;
	const file = target.files?.[0];

	if (file) {
		if (!file.type.startsWith("image/")) {
			alert("Veuillez sélecqtionner une image");
			return;
		}

		selectedFile.value = file;
		imagePreview.value = URL.createObjectURL(file);
		removeExistingImage.value = false;
	}
};

const removeSelectedImage = () => {
	if (imagePreview.value && selectedFile.value) {
		URL.revokeObjectURL(imagePreview.value);
	}
	selectedFile.value = null;
	imagePreview.value = null;
};

const markExistingImageForRemoval = () => {
	removeExistingImage.value = true;
	hasExistingImage.value = false;
	if (imagePreview.value) {
		URL.revokeObjectURL(imagePreview.value);
		imagePreview.value = null;
	}
};

onMounted(async () => {
	try {
		const doc = await props.storage.get(props.documentId);
		currentDoc.value = doc;
		bookName.value = doc.book_name || "";
		bookDescription.value = doc.book_description || "";
		bookCategory.value = doc.book_category || "";
		bookAuthor.value = doc.book_author || "";
		bookLikes.value = doc.book_likes || 0;

		// Charger l'image existante si elle es trouvée
		if (doc._attachments?.cover) {
			hasExistingImage.value = true;
			try {
				const blob = await props.storage.getAttachment(
					props.documentId,
					"cover",
				);
				imagePreview.value = URL.createObjectURL(blob);
			} catch (err) {
				console.error(
					"Erreur lors du chargement de l'image:",
					err,
				);
			}
		}
	} catch (err) {
		console.error("Erreur lors du chargement du document:", err);
		alert("Impossible de charger le document");
		emit("close");
	}
});

const handleSubmit = async (e: Event) => {
	e.preventDefault();

	if (!bookName.value || !bookDescription.value) {
		alert("Veuillez remplir tous les champs obligatoires");
		return;
	}

	if (!currentDoc.value) {
		alert("Document non chargé");
		return;
	}

	try {
		// Mettre à jour le document existant (garder _id et _rev)
		const updatedDoc = {
			...currentDoc.value,
			book_name: bookName.value,
			book_description: bookDescription.value,
			book_category: bookCategory.value,
			book_author: bookAuthor.value,
			book_likes: bookLikes.value,
			attributes: {
				...currentDoc.value.attributes,
				modification_date: new Date().toISOString(),
			},
		};

		let response = await props.storage.put(updatedDoc);
		console.log("Document modifié avec succès : ", response);

		// Supprimer l'image existante si demandé
		if (removeExistingImage.value) {
			response = await props.storage.removeAttachment(
				response.id,
				"cover",
				response.rev,
			);
			console.log("Image existante supprimée");
		}

		if (selectedFile.value) {
			await props.storage.putAttachment(
				response.id,
				"cover",
				response.rev,
				selectedFile.value,
				selectedFile.value.type,
			);
			console.log("Nouvelle image ajoutée");
		}

		emit("documentModified");
		emit("close");
	} catch (err: any) {
		console.error("Erreur lors de la modification du document : ", err);
		alert("Erreur lors de la modification du document");
	}
};

const handleClose = () => {
	emit("close");
};
</script>

<template>
	<div class="modal-overlay" @click.self="handleClose">
		<div class="modal-content">
			<div class="modal-header">
				<h2>Modifier le document</h2>
				<button class="close-btn" @click="handleClose">
					&times;
				</button>
			</div>
			<p v-if="props.isOnline === false" class="offline-warning">
				Vous êtes hors ligne — la modification sera appliquée
				localement et synchronisée lorsque la connexion
				reviendra.
			</p>
			<form @submit="handleSubmit">
				<div class="form-group">
					<label for="doc-name">Nom du livre:</label>
					<input
						type="text"
						id="doc-name"
						v-model="bookName"
						placeholder="Entrez le nom du livre"
						required
					/>
				</div>
				<div class="form-group">
					<label for="doc-description">Description:</label>
					<textarea
						id="doc-description"
						v-model="bookDescription"
						placeholder="Entrez la description"
						rows="6"
						required
					></textarea>
				</div>
				<div class="form-group">
					<label for="doc-category">Catégorie:</label>
					<input
						type="text"
						id="doc-category"
						v-model="bookCategory"
						placeholder="Ex: philo, psycho, roman, poésie..."
					/>
				</div>
				<div class="form-group">
					<label for="doc-author">Auteur:</label>
					<input
						type="text"
						id="doc-author"
						v-model="bookAuthor"
						placeholder="Entrez le nom de l'auteur"
					/>
				</div>
				<div class="form-group">
					<label for="doc-likes">Nombre de likes:</label>
					<input
						type="number"
						id="doc-likes"
						v-model="bookLikes"
						placeholder="Entrez le nombre de likes"
						min="0"
					/>
				</div>
				<div class="form-group">
					<label for="doc-image"
						>Image de couverture:</label
					>
					<div v-if="imagePreview" class="image-preview">
						<img
							:src="imagePreview"
							alt="Prévisualisation"
						/>
						<button
							v-if="hasExistingImage"
							type="button"
							class="btn-remove-image"
							@click="markExistingImageForRemoval"
						>
							Supprimer l'image
						</button>
						<button
							v-else-if="selectedFile"
							type="button"
							class="btn-remove-image"
							@click="removeSelectedImage"
						>
							Annuler la sélection
						</button>
					</div>
					<div
						v-if="removeExistingImage"
						class="image-removed-notice"
					>
						L'image sera supprimée lors de la
						sauvegarde.
					</div>
					<input
						type="file"
						id="doc-image"
						accept="image/*"
						@change="handleFileSelect"
					/>
					<small v-if="hasExistingImage || imagePreview">
						Sélectionner une nouvelle image remplacera
						l'existante.
					</small>
				</div>
				<div class="form-actions">
					<button
						type="button"
						class="btn-secondary"
						@click="handleClose"
					>
						Annuler
					</button>
					<button type="submit" class="btn-primary">
						Modifier
					</button>
				</div>
			</form>
		</div>
	</div>
</template>

<style scoped>
.modal-overlay {
	position: fixed;
	top: 0;
	left: 0;
	right: 0;
	bottom: 0;
	background-color: rgba(0, 0, 0, 0.5);
	display: flex;
	justify-content: center;
	align-items: center;
	z-index: 1000;
}

.modal-content {
	background-color: white;
	border-radius: 0.5rem;
	padding: 2rem;
	max-width: 500px;
	width: 90%;
	max-height: 90vh;
	overflow-y: auto;
	box-shadow: 0 10px 25px rgba(0, 0, 0, 0.2);
}

.modal-header {
	display: flex;
	justify-content: space-between;
	align-items: center;
	margin-bottom: 1.5rem;
}

.modal-header h2 {
	font-size: 1.5rem;
	font-weight: 600;
	color: #1a1a1a;
	margin: 0;
}

.close-btn {
	background: none;
	border: none;
	font-size: 2rem;
	color: #6b7280;
	cursor: pointer;
	padding: 0;
	width: 2rem;
	height: 2rem;
	display: flex;
	align-items: center;
	justify-content: center;
	border-radius: 0.25rem;
	transition: all 0.2s;
}

.close-btn:hover {
	background-color: #f3f4f6;
	color: #1a1a1a;
}

.offline-warning {
	background-color: #fef3c7;
	color: #92400e;
	padding: 0.75rem;
	border-radius: 0.375rem;
	margin-bottom: 1rem;
	font-size: 0.875rem;
}

.form-group {
	margin-bottom: 1.25rem;
}

.form-group label {
	display: block;
	margin-bottom: 0.5rem;
	font-weight: 500;
	color: #374151;
	font-size: 0.875rem;
}

.form-group input,
.form-group textarea {
	width: 100%;
	padding: 0.625rem;
	border: 1px solid #d1d5db;
	border-radius: 0.375rem;
	font-size: 0.875rem;
	font-family: inherit;
	transition: all 0.2s;
}

.form-group input:focus,
.form-group textarea:focus {
	outline: none;
	border-color: #6366f1;
	box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.1);
}

.form-group textarea {
	resize: vertical;
	min-height: 100px;
}

.form-actions {
	display: flex;
	justify-content: flex-end;
	gap: 0.75rem;
	margin-top: 1.5rem;
}

.btn-primary,
.btn-secondary {
	padding: 0.625rem 1.25rem;
	border-radius: 0.375rem;
	font-size: 0.875rem;
	font-weight: 500;
	cursor: pointer;
	transition: all 0.2s;
	border: none;
}

.btn-primary {
	background-color: #6366f1;
	color: white;
}

.btn-primary:hover {
	background-color: #4f46e5;
}

.btn-secondary {
	background-color: white;
	color: #374151;
	border: 1px solid #d1d5db;
}

.btn-secondary:hover {
	background-color: #f9fafb;
}

.btn-primary:active,
.btn-secondary:active {
	transform: scale(0.98);
}

.image-preview {
	margin: 0.75rem 0;
	text-align: center;
}

.image-preview img {
	max-width: 100%;
	max-height: 200px;
	border-radius: 0.375rem;
	border: 1px solid #d1d5db;
	object-fit: contain;
}

.btn-remove-image {
	display: block;
	margin: 0.5rem auto 0;
	padding: 0.375rem 0.75rem;
	background-color: #dc2626;
	color: white;
	border: none;
	border-radius: 0.375rem;
	font-size: 0.75rem;
	cursor: pointer;
	transition: all 0.2s;
}

.btn-remove-image:hover {
	background-color: #b91c1c;
}

.image-removed-notice {
	background-color: #fef3c7;
	color: #92400e;
	padding: 0.5rem;
	border-radius: 0.375rem;
	font-size: 0.75rem;
	margin: 0.5rem 0;
	text-align: center;
}

.form-group small {
	display: block;
	margin-top: 0.25rem;
	color: #6b7280;
	font-size: 0.75rem;
}
</style>
