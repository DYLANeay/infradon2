<script setup lang="ts">
import { ref, onMounted, onUnmounted } from "vue";

/**
 *
 * Pourquoi pas utiliser allDocs avec include_docs ?
 *    → allDocs charge TOUS les documents en mémoire, même ceux qu'on n'affiche pas (donc useless si on veut 10/10 ou que l'utilisateur ne scrolle pas jusqu'en bas)
 *
 * .find avec limit comme options
 *    → Charge uniquement les N documents demandés
 *    → skip permet la pagination sans recharger les documents précédents
 *
 * Pourquoi un index composite [type, book_id, creation_date] pour les commentaires ?
 *    → Permet de récupérer le dernier commentaire en une seule  (trié juste direct)
 *    → Évite de charger tous les commentaires puis filtrer côté client
 *
 * 4. Lazy loading avec Intersection Observer :
 *    → Charge les données uniquement quand l'utilisateur scrolle
 *    → Réduit la charge initiale et les échanges réseau
 */

interface Book {
	_id?: string;
	_rev?: string;
	book_name: string;
	book_description: string;
	review_title?: string;
	book_category?: string;
	book_author?: string;
	book_likes: number;
	attributes: {
		creation_date: any;
		modification_date?: string;
	};
	_attachments?: any;
	coverUrl?: string;
}

interface Comment {
	_id?: string;
	_rev?: string;
	type: "comment";
	book_id: string;
	content: string;
	author: string;
	attributes: {
		creation_date: string;
	};
}

interface BookWithComment extends Book {
	lastComment: Comment | null;
	showAllComments: boolean;
	allComments: Comment[];
	loadingComments: boolean;
}

const props = defineProps<{
	storage: any;
	commentsStorage: any;
}>();

// État
const books = ref<BookWithComment[]>([]);
const isLoading = ref(false);
const hasMore = ref(true);
const currentSkip = ref(0);
const LIMIT = 10;

// Référence pour l'élément sentinel qui permet donc de savoir à quel "hauteur" de scroll nous sommes
const sentinelRef = ref<HTMLElement | null>(null);
let observer: IntersectionObserver | null = null;

/**
 * Charge les N livres (dépendant de la vlaeur de la constante) les plus likés avec pagination
 * Utilise .find() avec l'index sur book_likes pour efficacité
 */
const loadBooks = async () => {
	if (isLoading.value || !hasMore.value) return;

	isLoading.value = true;

	try {
		const result = await props.storage.find({
			selector: {
				book_likes: { $gte: 0 },
			},
			sort: [{ book_likes: "desc" }],
			limit: LIMIT,
			skip: currentSkip.value,
		});

		const newBooks = result.docs as Book[];

		// Si moins de LIMIT résultats, on a tout chargé, donc rien de plus à charger potentiellement
		if (newBooks.length < LIMIT) {
			hasMore.value = false;
		}

		// Pour chaque livre, récupérer le dernier commentaire
		const booksWithComments: BookWithComment[] = await Promise.all(
			newBooks.map(async (book) => {
				const lastComment = await getLastComment(book._id!);
				return {
					...book,
					lastComment,
					showAllComments: false,
					allComments: [],
					loadingComments: false,
				};
			}),
		);

		// Ajouter les nouveaux livres à la liste existante
		books.value = [...books.value, ...booksWithComments];
		currentSkip.value += newBooks.length;
	} catch (error) {
		console.error("Erreur lors du chargement des livres:", error);
	} finally {
		isLoading.value = false;
	}
};

/**
 * Récupère le dernier commentaire d'un livre
 * Optimisation : utilise .find() avec limit: 1 et tri décroissant
 */
const getLastComment = async (bookId: string): Promise<Comment | null> => {
	try {
		const result = await props.commentsStorage.find({
			selector: {
				type: "comment",
				book_id: bookId,
			},
			sort: [{ "attributes.creation_date": "desc" }],
			limit: 1, // On ne veut que le dernier et qui plus est le plus récent (fais en dessus=)
		});

		return result.docs.length > 0 ? result.docs[0] : null;
	} catch (error) {
		// Si l'index n'existe pas encore, fallback sur allDocs filtré
		// (moins efficace mais fonctionnel)
		console.warn("Index non trouvé, fallback sur allDocs:", error);
		const allDocs = await props.commentsStorage.allDocs({
			include_docs: true,
		});
		const comments = allDocs.rows
			.map((row: any) => row.doc)
			.filter(
				(doc: any) =>
					doc.type === "comment" && doc.book_id === bookId,
			)
			.sort(
				(a: Comment, b: Comment) =>
					new Date(b.attributes.creation_date).getTime() -
					new Date(a.attributes.creation_date).getTime(),
			);
		return comments.length > 0 ? comments[0] : null;
	}
};

/**
 * Charge tous les commentaires d'un livre (pour le toggle)
 */
const loadAllComments = async (book: BookWithComment) => {
	if (book.loadingComments) return;

	book.loadingComments = true;

	try {
		const result = await props.commentsStorage.find({
			selector: {
				type: "comment",
				book_id: book._id,
			},
			sort: [{ "attributes.creation_date": "desc" }],
		});

		book.allComments = result.docs;
	} catch (error) {
		// Fallback si index non disponible
		const allDocs = await props.commentsStorage.allDocs({
			include_docs: true,
		});
		book.allComments = allDocs.rows
			.map((row: any) => row.doc)
			.filter(
				(doc: any) =>
					doc.type === "comment" &&
					doc.book_id === book._id,
			)
			.sort(
				(a: Comment, b: Comment) =>
					new Date(b.attributes.creation_date).getTime() -
					new Date(a.attributes.creation_date).getTime(),
			);
	} finally {
		book.loadingComments = false;
	}
};

/**
 * Toggle l'affichage de tous les commentaires
 */
const toggleAllComments = async (book: BookWithComment) => {
	if (!book.showAllComments && book.allComments.length === 0) {
		await loadAllComments(book);
	}
	book.showAllComments = !book.showAllComments;
};

/**
 * Formate une date pour l'affichage
 */
const formatDate = (dateString: string): string => {
	const date = new Date(dateString);
	return date.toLocaleDateString("fr-FR", {
		day: "numeric",
		month: "short",
		year: "numeric",
		hour: "2-digit",
		minute: "2-digit",
	});
};

/**
 * Configuration de l'Intersection Observer pour le lazy loading
 * Quand l'élément sentinel devient visible → charger plus de livres
 */
const setupIntersectionObserver = () => {
	observer = new IntersectionObserver(
		(entries) => {
			const entry = entries[0];
			// Si l'élément est visible et qu'on n'est pas déjà en train de charger
			if (
				entry.isIntersecting &&
				!isLoading.value &&
				hasMore.value
			) {
				loadBooks();
			}
		},
		{
			// Déclencher un peu avant que l'élément soit visible pour éviter des lags
			rootMargin: "100px",
		},
	);

	if (sentinelRef.value) {
		observer.observe(sentinelRef.value);
	}
};

onMounted(async () => {
	await loadBooks();
	setupIntersectionObserver();
});

onUnmounted(() => {
	// Nettoyer l'observer pour qu'il ne prenne pas de mémoire inutilement en mémoire
	if (observer) {
		observer.disconnect();
	}
});
</script>

<template>
	<div>
		<h3>Top des livres les plus likés</h3>

		<template v-for="book in books" :key="book._id">
			<article v-if="book.book_name">
				<h2>{{ book.book_name }}</h2>
				<p v-if="book.book_category">
					<strong>Catégorie:</strong>
					{{ book.book_category }}
				</p>
				<p v-if="book.book_author">
					<strong>Auteur:</strong> {{ book.book_author }}
				</p>
				<h4>{{ book.book_description }}</h4>
				<p>
					{{ book.book_likes ?? 0 }}
					{{
						(book.book_likes ?? 0) > 1
							? "likes"
							: "like"
					}}
				</p>

				<!-- Dernier commentaire -->
				<div
					class="last-comment-section"
					v-if="book.lastComment"
				>
					<h3>Dernier commentaire</h3>
					<p>{{ book.lastComment.content }}</p>
					<p class="comment-meta">
						— {{ book.lastComment.author }},
						{{
							formatDate(
								book.lastComment.attributes
									.creation_date,
							)
						}}
					</p>
				</div>
				<p v-else><em>Aucun commentaire pour ce livre</em></p>

				<!-- Bouton voir tous les commentaires -->
				<button
					@click="toggleAllComments(book)"
					:disabled="book.loadingComments"
				>
					<span v-if="book.loadingComments"
						>Chargement...</span
					>
					<span v-else-if="book.showAllComments"
						>Masquer les commentaires</span
					>
					<span v-else>Voir tous les commentaires</span>
				</button>

				<!-- Liste de tous les commentaires (toggle) -->
				<div v-if="book.showAllComments" class="all-comments">
					<p v-if="book.allComments.length === 0">
						Aucun commentaire
					</p>
					<div
						v-for="comment in book.allComments"
						:key="comment._id"
						class="comment-item"
					>
						<p>{{ comment.content }}</p>
						<p class="comment-meta">
							{{ comment.author }} -
							{{
								formatDate(
									comment.attributes
										.creation_date,
								)
							}}
						</p>
					</div>
				</div>
			</article>
		</template>

		<div ref="sentinelRef" class="sentinel">
			<p v-if="isLoading">Chargement des livres suivants...</p>
			<p v-else-if="!hasMore">Fin de la liste</p>
		</div>
	</div>
</template>

<style scoped>
div > h3 {
	font-size: 1.25rem;
	font-weight: 600;
	color: #1a1a1a;
	margin-bottom: 1rem;
	margin-top: 2rem;
}

article {
	background-color: white;
	border: 1px solid #e5e7eb;
	border-radius: 0.5rem;
	padding: 1.5rem;
	margin-bottom: 1rem;
	box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

article h2 {
	font-size: 1.5rem;
	font-weight: 600;
	color: #1a1a1a;
	margin-bottom: 0.5rem;
}

article h3 {
	font-size: 1.125rem;
	font-weight: 500;
	color: #374151;
	margin: 0.75rem 0;
}

article h4 {
	font-size: 1rem;
	font-weight: 400;
	color: #6b7280;
	margin: 0.75rem 0;
	line-height: 1.6;
}

article p {
	color: #6b7280;
	font-size: 0.875rem;
	margin: 0.5rem 0;
}

article p strong {
	color: #374151;
	font-weight: 600;
}

button {
	padding: 0.5rem 1rem;
	border: 1px solid #d1d5db;
	background-color: white;
	color: #374151;
	border-radius: 0.375rem;
	cursor: pointer;
	font-size: 0.875rem;
	transition: all 0.2s;
	margin-right: 0.5rem;
	margin-top: 0.5rem;
}

button:hover {
	background-color: #f9fafb;
	border-color: #9ca3af;
}

button:active {
	transform: scale(0.98);
}

button:disabled {
	opacity: 0.6;
	cursor: not-allowed;
}

.last-comment-section {
	margin-top: 1rem;
	padding: 1rem;
	background-color: #f9fafb;
	border-left: 3px solid #6366f1;
	border-radius: 0.25rem;
}

.last-comment-section h3 {
	margin: 0 0 0.5rem 0;
	font-size: 0.875rem;
}

.last-comment-section p {
	margin: 0.25rem 0;
}

.comment-meta {
	font-size: 0.75rem;
	color: #9ca3af;
}

/* Tous les commentaires */
.all-comments {
	margin-top: 1rem;
	padding: 1rem;
	background-color: #f3f4f6;
	border-radius: 0.375rem;
}

.comment-item {
	padding: 0.75rem;
	background: white;
	border: 1px solid #e5e7eb;
	border-radius: 0.375rem;
	margin-bottom: 0.5rem;
}

.comment-item:last-child {
	margin-bottom: 0;
}

.comment-item p {
	margin: 0;
}

.sentinel {
	padding: 1rem;
	text-align: center;
}

.sentinel p {
	color: #6b7280;
	font-style: italic;
}
</style>
