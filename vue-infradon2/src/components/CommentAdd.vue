<script setup lang="ts">
declare interface Comment {
	content: string;
	author: string;
	book_id: string;
}
interface Props {
	storage: any;
	isOnline?: boolean;
}
const props = defineProps<Props>();
const emit = defineEmits<{
	close: [];
	commentAdded: [];
}>();
</script>

<template>
	<div></div>
</template>

<style scoped></style>
