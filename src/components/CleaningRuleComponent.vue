<script setup lang="ts">
import { use_cleaning_store, type CleaningRuleProperties } from "../stores/cleaning";

const props = defineProps<{
	hostname?: string;
} & Partial<CleaningRuleProperties>>();

const cleaning_store = use_cleaning_store();

function update_hostname(event: FocusEvent) {
	const input = event.target as HTMLInputElement;
	input.setCustomValidity("");
	if (!input.reportValidity()) {
		return;
	}
	if (input.value.length > 0) {
		const rule = cleaning_store.rules[input.value];
		if (rule === undefined) {
			cleaning_store.rules[input.value] = Object.assign({}, cleaning_store.ruledefault);
			cleaning_store.updated = true;
			input.value = "";
		} else if (props.hostname !== input.value) {
			input.setCustomValidity(`Duplicate hostname in rules is not allowed.`);
			input.reportValidity();
			return;
		}
	}
	if (props.hostname !== undefined && props.hostname !== input.value) {
		delete cleaning_store.rules[props.hostname];
		cleaning_store.updated = true;
	}
}

function update_subdomains(event: Event) {
	const checkbox = event.target as HTMLInputElement;
	if (props.hostname === undefined) {
		cleaning_store.ruledefault.subdomains = checkbox.checked;
		cleaning_store.updated = true;
		return;
	}
	const rule = cleaning_store.rules[props.hostname];
	if (rule === undefined) {
		console.error(`History rule for ${props.hostname} is undefined.`);
		return;
	}
	rule.subdomains = checkbox.checked;
	cleaning_store.updated = true;
}

function update_bookmarks(event: Event) {
	const checkbox = event.target as HTMLInputElement;
	if (props.hostname === undefined) {
		cleaning_store.ruledefault.bookmarks = checkbox.checked;
		cleaning_store.updated = true;
		return;
	}
	const rule = cleaning_store.rules[props.hostname];
	if (rule === undefined) {
		console.error(`History rule for ${props.hostname} is undefined.`);
		return;
	}
	rule.bookmarks = checkbox.checked;
	cleaning_store.updated = true;
}

function update_history(event: Event) {
	const checkbox = event.target as HTMLInputElement;
	if (props.hostname === undefined) {
		cleaning_store.ruledefault.history = checkbox.checked;
		cleaning_store.updated = true;
		return;
	}
	const rule = cleaning_store.rules[props.hostname];
	if (rule === undefined) {
		console.error(`History rule for ${props.hostname} is undefined.`);
		return;
	}
	rule.history = checkbox.checked;
	cleaning_store.updated = true;
}
</script>

<template>
	<li class="list-row join">
		<label class="input validator join-item w-full max-w-full">
			Hostname
			<input type="text" pattern="(?:[\p{L}\p{N}\-]+\.)+[\p{L}\p{N}]{2,}" placeholder="example.com" autocomplete="off" spellcheck="false" :value="hostname" @focusout="update_hostname" />
		</label>
		<label class="label join-item">
			<input type="checkbox" class="toggle" :checked="subdomains" @change="update_subdomains" />
			Subdomains
		</label>
		<label class="label join-item">
			<input type="checkbox" class="toggle" :checked="bookmarks" @change="update_bookmarks" />
			Bookmarks
		</label>
		<label class="label join-item">
			<input type="checkbox" class="toggle" :checked="history" @change="update_history" />
			History
		</label>
	</li>
</template>
