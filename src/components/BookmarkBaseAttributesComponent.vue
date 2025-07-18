<script setup lang="ts">
import { faFileImport, faHashtag, faQuoteLeft } from "@fortawesome/free-solid-svg-icons";
import { FontAwesomeIcon } from "@fortawesome/vue-fontawesome";
import type { Bookmark } from "../stores/bookmarks";

defineProps<Bookmark>();
</script>

<template>
	<fieldset class="fieldset">
		<legend class="fieldset-legend">Basic</legend>
		<label class="input w-full max-w-full">
			<FontAwesomeIcon :icon="faHashtag" />
			<input :id="`bookmark-${id}-id`" type="text" class="grow" autocomplete="off" readonly :value="id" />
		</label>
		<label class="input w-full max-w-full">
			<FontAwesomeIcon :icon="faQuoteLeft" />
			<input :id="`bookmark-${id}-title`" type="text" class="grow" :readonly="immutable" autocomplete="off" :value="title" />
		</label>
		<label class="input w-full max-w-full">
			<FontAwesomeIcon :icon="faFileImport" />
			<input :id="`bookmark-${id}-added`" type="text" class="grow" readonly autocomplete="off" :value="added.toLocaleString()" />
		</label>
	</fieldset>
</template>
