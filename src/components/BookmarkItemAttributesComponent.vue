<script setup lang="ts">
import { faClockRotateLeft, faLink } from "@fortawesome/free-solid-svg-icons";
import { FontAwesomeIcon } from "@fortawesome/vue-fontawesome";
import type { BookmarkItem } from "../stores/bookmarks";

defineProps<BookmarkItem>();
</script>

<template>
	<fieldset class="fieldset">
		<legend class="fieldset-legend">Item</legend>
		<label class="input w-full max-w-full">
			<FontAwesomeIcon :icon="faClockRotateLeft" />
			<input :id="`bookmark-${id}-last-used`" type="text" class="grow" readonly autocomplete="off" :value="last_used.toLocaleString()" />
		</label>
		<label class="input w-full max-w-full">
			<FontAwesomeIcon :icon="faLink" />
			<input :id="`bookmark-${id}-url`" type="url" class="grow" :readonly="immutable" autocomplete="off" :value="url" />
		</label>
	</fieldset>
</template>
