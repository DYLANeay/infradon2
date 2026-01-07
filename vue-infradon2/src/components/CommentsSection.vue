<script setup lang="ts">
import { ref, onMounted } from "vue";

interface Comment {
	_id?: string;
	_rev?: string;
	type: "comment";
	book_id: string; // Foreign key vers le livre
	content: string;
	author: string;
	attributes: {
		creation_date: string;
	};
}

interface Props {
	storage: any;
	bookId: string;
	isOnline?: boolean;
}

const props = defineProps<Props>();
const emit = defineEmits<{
	commentAdded: [];
	commentDeleted: [];
	commentUpdated: [];
}>();

const comments = ref<Comment[]>([]);
const newCommentContent = ref("");
const newCommentAuthor = ref("");
const showAddForm = ref(false);

// état pour l'édition de commentaire
const editingCommentId = ref<string | null>(null);
const editContent = ref("");
const editAuthor = ref("");

const loadComments = async () => {
	try {
		const allDocs = await props.storage.allDocs({ include_docs: true });
		comments.value = allDocs.rows
			.map((row: any) => row.doc)
			.filter(
				(doc: any) =>
					doc.type === "comment" &&
					doc.book_id === props.bookId,
			)
			.sort(
				(a: Comment, b: Comment) =>
					new Date(b.attributes.creation_date).getTime() -
					new Date(a.attributes.creation_date).getTime(),
			);
	} catch (err) {
		console.error("Erreur lors du chargement des commentaires:", err);
	}
};

// Ajouter un commentaire
const addComment = async () => {
	if (!newCommentContent.value.trim() || !newCommentAuthor.value.trim()) {
		alert("Veuillez remplir tous les champs");
		return;
	}

	const newComment: Comment = {
		type: "comment",
		book_id: props.bookId,
		content: newCommentContent.value.trim(),
		author: newCommentAuthor.value.trim(),
		attributes: {
			creation_date: new Date().toISOString(),
		},
	};

	try {
		await props.storage.post(newComment);
		console.log("Commentaire ajouté avec succès");
		newCommentContent.value = "";
		newCommentAuthor.value = "";
		showAddForm.value = false;
		await loadComments();
		emit("commentAdded");
	} catch (err) {
		console.error("Erreur lors de l'ajout du commentaire:", err);
		alert("Erreur lors de l'ajout du commentaire");
	}
};

// Supprimer un commentaire
const deleteComment = async (commentId: string) => {
	if (!confirm("Voulez-vous vraiment supprimer ce commentaire ?")) {
		return;
	}

	try {
		const doc = await props.storage.get(commentId);
		await props.storage.remove(doc);
		console.log("Commentaire supprimé avec succès");
		await loadComments();
		emit("commentDeleted");
	} catch (err) {
		console.error("Erreur lors de la suppression du commentaire:", err);
		alert("Erreur lors de la suppression du commentaire");
	}
};

//partie pour modif un commentaire
const startEdit = (comment: Comment) => {
	editingCommentId.value = comment._id!; //! qui assure à TS que la valeur de comment._id n'est pas null ou undefined
	editContent.value = comment.content;
	editAuthor.value = comment.author;
};

const cancelEdit = () => {
	editingCommentId.value = null;
	editContent.value = "";
	editAuthor.value = "";
};

const saveEdit = async () => {
	if (!editContent.value.trim() || !editAuthor.value.trim()) {
		alert("Veuillez remplir tous les champs");
		return;
	}

	try {
		const doc = await props.storage.get(editingCommentId.value);
		const updatedComment = {
			...doc,
			content: editContent.value.trim(),
			author: editAuthor.value.trim(),
		};
		await props.storage.put(updatedComment);
		console.log("Commentaire modifié avec succès");
		cancelEdit();
		await loadComments();
		emit("commentUpdated");
	} catch (err) {
		console.error(
			"Erreur lors de la modification du commentaire:",
			err,
		);
	}
};

onMounted(() => {
	loadComments();
});
</script>

<template>
	<div class="comments-section">
		<div class="comments-header">
			<h4>Commentaires ({{ comments.length }})</h4>
			<button
				@click="showAddForm = !showAddForm"
				class="btn-add-comment"
			>
				{{
					showAddForm
						? "Annuler"
						: "+ Ajouter un commentaire"
				}}
			</button>
		</div>

		<!-- Formulaire d'ajout de commentaire -->
		<div v-if="showAddForm" class="add-comment-form">
			<div class="form-group">
				<input
					type="text"
					v-model="newCommentAuthor"
					placeholder="Votre nom"
					class="comment-input"
				/>
			</div>
			<div class="form-group">
				<textarea
					v-model="newCommentContent"
					placeholder="Votre commentaire"
					rows="3"
					class="comment-textarea"
				></textarea>
			</div>
			<div class="form-actions">
				<button @click="addComment" class="btn-primary">
					Publier
				</button>
				<button
					@click="showAddForm = false"
					class="btn-secondary"
				>
					Annuler
				</button>
			</div>
		</div>

		<!-- Liste des commentaires -->
		<div class="comments-list">
			<div
				v-if="comments.length === 0 && !showAddForm"
				class="no-comments"
			>
				Aucun commentaire pour le moment. Soyez le premier à
				commenter !
			</div>
			<div
				v-for="comment in comments"
				:key="comment._id"
				class="comment-item"
			>
				<!-- Mode édition -->
				<div
					v-if="editingCommentId === comment._id"
					class="edit-comment-form"
				>
					<div class="form-group">
						<input
							type="text"
							v-model="editAuthor"
							placeholder="Votre nom"
							class="comment-input"
						/>
					</div>
					<div class="form-group">
						<textarea
							v-model="editContent"
							placeholder="Votre commentaire"
							rows="3"
							class="comment-textarea"
						></textarea>
					</div>
					<div class="form-actions">
						<button
							@click="saveEdit"
							class="btn-primary"
						>
							Sauvegarder
						</button>
						<button
							@click="cancelEdit"
							class="btn-secondary"
						>
							Annuler
						</button>
					</div>
				</div>
				<!-- Mode affichage -->
				<template v-else>
					<div class="comment-header">
						<strong>{{ comment.author }}</strong>
						<span class="comment-date">
							{{
								new Date(
									comment.attributes.creation_date,
								).toLocaleDateString("fr-FR")
							}}
						</span>
					</div>
					<p class="comment-content">
						{{ comment.content }}
					</p>
					<div class="comment-actions">
						<button
							@click="startEdit(comment)"
							class="btn-edit-comment"
						>
							Modifier
						</button>
						<button
							@click="deleteComment(comment._id!)"
							class="btn-delete-comment"
						>
							Supprimer
						</button>
					</div>
				</template>
			</div>
		</div>
	</div>
</template>

<style scoped>
.comments-section {
	margin-top: 1.5rem;
	padding-top: 1.5rem;
	border-top: 1px solid #e5e7eb;
}

.comments-header {
	display: flex;
	justify-content: space-between;
	align-items: center;
	margin-bottom: 1rem;
}

.comments-header h4 {
	margin: 0;
	font-size: 1rem;
	font-weight: 600;
	color: #374151;
}

.btn-add-comment {
	padding: 0.5rem 0.75rem;
	background-color: #6366f1;
	color: white;
	border: none;
	border-radius: 0.375rem;
	font-size: 0.875rem;
	cursor: pointer;
	transition: all 0.2s;
}

.btn-add-comment:hover {
	background-color: #4f46e5;
}

.add-comment-form,
.edit-comment-form {
	background-color: #f9fafb;
	padding: 1rem;
	border-radius: 0.375rem;
	margin-bottom: 1rem;
}

.form-group {
	margin-bottom: 0.75rem;
}

.comment-input,
.comment-textarea {
	width: 100%;
	padding: 0.5rem;
	border: 1px solid #d1d5db;
	border-radius: 0.375rem;
	font-size: 0.875rem;
	font-family: inherit;
}

.comment-input:focus,
.comment-textarea:focus {
	outline: none;
	border-color: #6366f1;
	box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.1);
}

.comment-textarea {
	resize: vertical;
}

.form-actions {
	display: flex;
	gap: 0.5rem;
}

.btn-primary,
.btn-secondary {
	padding: 0.5rem 1rem;
	border-radius: 0.375rem;
	font-size: 0.875rem;
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

.comments-list {
	display: flex;
	flex-direction: column;
	gap: 1rem;
}

.no-comments {
	padding: 1rem;
	text-align: center;
	color: #9ca3af;
	font-size: 0.875rem;
	font-style: italic;
}

.comment-item {
	background-color: #f9fafb;
	padding: 0.75rem;
	border-radius: 0.375rem;
	border: 1px solid #e5e7eb;
}

.comment-header {
	display: flex;
	justify-content: space-between;
	align-items: center;
	margin-bottom: 0.5rem;
}

.comment-header strong {
	color: #1f2937;
	font-size: 0.875rem;
}

.comment-date {
	color: #9ca3af;
	font-size: 0.75rem;
}

.comment-content {
	color: #374151;
	font-size: 0.875rem;
	margin: 0.5rem 0;
	line-height: 1.5;
}

.comment-actions {
	display: flex;
	gap: 0.5rem;
}

.btn-edit-comment {
	padding: 0.25rem 0.5rem;
	background-color: transparent;
	color: #6366f1;
	border: 1px solid #6366f1;
	border-radius: 0.25rem;
	font-size: 0.75rem;
	cursor: pointer;
	transition: all 0.2s;
}

.btn-edit-comment:hover {
	background-color: #6366f1;
	color: white;
}

.btn-delete-comment {
	padding: 0.25rem 0.5rem;
	background-color: transparent;
	color: #dc2626;
	border: 1px solid #dc2626;
	border-radius: 0.25rem;
	font-size: 0.75rem;
	cursor: pointer;
	transition: all 0.2s;
}

.btn-delete-comment:hover {
	background-color: #dc2626;
	color: white;
}
</style>
