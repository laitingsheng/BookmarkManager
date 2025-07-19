<script setup lang="ts">
import BookmarkTree from "./components/BookmarkTree.vue";
import { use_bookmarks_store } from "./stores/bookmarks";
import { use_preferences_store } from "./stores/preferences";

const bookmarks_store = use_bookmarks_store();
const preferences_store = use_preferences_store();

chrome.storage.sync.get<Record<string, boolean>>(
	{
		folderonly: true,
	},
	({ folderonly }) => {
		if (chrome.runtime.lastError !== undefined) {
			console.error(`chrome.storage.sync.get: ${chrome.runtime.lastError.message}`);
			return;
		}
		preferences_store.folderonly = folderonly;
	},
);

function switch_folderonly() {
	preferences_store.folderonly = !preferences_store.folderonly;
	chrome.storage.sync.set(
		{
			folderonly: preferences_store.folderonly,
		},
		() => {
			if (chrome.runtime.lastError !== undefined) {
				console.error(`chrome.storage.sync.set: ${chrome.runtime.lastError.message}`);
				return;
			}
		},
	);
}

chrome.bookmarks.getTree((nodes) => {
	if (chrome.runtime.lastError !== undefined) {
		console.error(`chrome.bookmarks.getTree: ${chrome.runtime.lastError.message}`);
		return;
	}
	if (nodes.length !== 1) {
		console.error("chrome.bookmarks.getTree: Bookmark root node is not a single node.");
		return;
	}
	const root = nodes[0];
	if (root.children?.length !== 2) {
		console.error("chrome.bookmarks.getTree: Bookmark should have exactly two top-level nodes.");
		return;
	}
	const parent = bookmarks_store.traverse(root.children[0]);
	if (parent.folder === true) {
		bookmarks_store.parent = parent;
	} else {
		console.error("chrome.bookmarks.getTree: Parent node is not a folder.");
	}
	const others = bookmarks_store.traverse(root.children[1]);
	if (others.folder === true) {
		bookmarks_store.others = others;
	} else {
		console.error("chrome.bookmarks.getTree: Others node is not a folder.");
	}
});
</script>

<template>
	<header class="bg-base-100 navbar shadow-sm">
		<div class="navbar-start">
			<a class="btn btn-ghost text-xl">Bookmark Manager</a>
		</div>
		<div class="navbar-end">
			<label class="label mx-2">
				<input type="checkbox" :checked="preferences_store.folderonly" class="toggle toggle-primary" @click="switch_folderonly()" />
				Folder Only
			</label>
		</div>
	</header>
	<main class="bg-base-100 block px-8">
		<div class="card bg-base-200">
			<div class="card-body">
				<h2 class="card-title">Bookmark Tree</h2>
				<ul class="menu w-full">
					<BookmarkTree v-if="bookmarks_store.parent" v-bind="bookmarks_store.parent" />
					<BookmarkTree v-if="bookmarks_store.others" v-bind="bookmarks_store.others" />
				</ul>
				<div class="card-actions justify-end">
					<button type="button" class="btn btn-primary" disabled>Save</button>
				</div>
			</div>
		</div>
	</main>
</template>
