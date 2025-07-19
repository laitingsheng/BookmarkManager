<script setup lang="ts">
import BookmarkTree from "./components/BookmarkTree.vue";
import { use_bookmarks_store, type Bookmark } from "./stores/bookmarks";
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

chrome.bookmarks.onChanged.addListener((id, info) => {
	if (chrome.runtime.lastError !== undefined) {
		console.error(`chrome.bookmarks.onChanged: ${chrome.runtime.lastError.message}`);
		return;
	}
	const bookmark = bookmarks_store.bookmarks[id];
	if (bookmark === undefined) {
		console.error(`Detected bookmark change for unknown id ${id}.`);
		return;
	}
	bookmark.title = info.title;
	if (info.url !== undefined) {
		if (bookmark.folder) {
			console.error(`chrome.bookmarks.onChanged: URL is assigned to folder #${id}.`);
			return;
		}
		bookmark.url = info.url;
	}
});

chrome.bookmarks.onChildrenReordered.addListener((id, info) => {
	if (chrome.runtime.lastError !== undefined) {
		console.error(`chrome.bookmarks.onChildrenReordered: ${chrome.runtime.lastError.message}`);
		return;
	}
	const bookmark = bookmarks_store.bookmarks[id];
	if (bookmark === undefined) {
		console.error(`chrome.bookmarks.onChildrenReordered: Detected children reordering for unknown id ${id}.`);
		return;
	}
	if (!bookmark.folder) {
		console.error(`chrome.bookmarks.onChildrenReordered: Bookmark #${id} is not a folder.`);
		return;
	}
	const children = new Array<Bookmark>();
	for (const cid of info.childIds) {
		const child = bookmarks_store.bookmarks[cid];
		if (child === undefined) {
			console.error(`chrome.bookmarks.onChildrenReordered: Bookmark #${id} has an unknown child id ${cid}.`);
			continue;
		}
		children.push(child);
	}
	bookmark.children = children;
});

chrome.bookmarks.onCreated.addListener((id, node) => {
	if (chrome.runtime.lastError !== undefined) {
		console.error(`chrome.bookmarks.onCreated: ${chrome.runtime.lastError.message}`);
		return;
	}
	if (node.parentId === undefined) {
		console.error(`chrome.bookmarks.onCreated: Unexpected bookmark #${id} has no valid parent.`);
		return;
	}
	const parent = bookmarks_store.bookmarks[node.parentId];
	if (parent === undefined) {
		console.error(`chrome.bookmarks.onCreated: Parent #${node.parentId} of bookmark #${id} is not found.`);
		return;
	}
	if (!parent.folder) {
		console.error(`chrome.bookmarks.onCreated: Parent #${node.parentId} of bookmark #${id} is not a folder.`);
		return;
	}
	parent.children.push(bookmarks_store.traverse(node));
});

chrome.bookmarks.onMoved.addListener((id, info) => {
	if (chrome.runtime.lastError !== undefined) {
		console.error(`chrome.bookmarks.onMoved: ${chrome.runtime.lastError.message}`);
		return;
	}
	const source = bookmarks_store.bookmarks[info.oldParentId];
	if (source === undefined) {
		console.error(`chrome.bookmarks.onMoved: Detected move of bookmark #${id} from unknown folder #${info.oldParentId}.`);
		return;
	}
	if (!source.folder) {
		console.error(`chrome.bookmarks.onMoved: Source #${info.oldParentId} of moved bookmark #${id} is not a folder.`);
		return;
	}
	const target = bookmarks_store.bookmarks[info.parentId];
	if (target === undefined) {
		console.error(`chrome.bookmarks.onMoved: Detected move of bookmark #${id} to unknown folder #${info.parentId}.`);
		return;
	}
	if (!target.folder) {
		console.error(`chrome.bookmarks.onMoved: Target #${info.parentId} of moved bookmark #${id} is not a folder.`);
		return;
	}
	if (info.oldIndex < source.children.length && source.children[info.oldIndex]?.id === id && info.index <= target.children.length) {
		const bookmark = bookmarks_store.bookmarks[id];
		if (bookmark === undefined) {
			console.error(`chrome.bookmarks.onMoved: Detected move of unknown bookmark #${id}.`);
			return;
		}
		source.children.splice(info.oldIndex, 1);
		target.children.splice(info.index, 0, bookmark);
	}
});

chrome.bookmarks.onRemoved.addListener((id, info) => {
	if (chrome.runtime.lastError !== undefined) {
		console.error(`chrome.bookmarks.onRemoved: ${chrome.runtime.lastError.message}`);
		return;
	}
	const parent = bookmarks_store.bookmarks[info.parentId];
	if (parent === undefined) {
		console.error(`chrome.bookmarks.onRemoved: Parent #${info.parentId} of removed bookmark #${id} is not found.`);
		return;
	}
	if (!parent.folder) {
		console.error(`chrome.bookmarks.onRemoved: Parent #${info.parentId} of removed bookmark #${id} is not a folder.`);
		return;
	}
	if (info.index < parent.children.length && parent.children[info.index]?.id === id) {
		parent.children.splice(info.index, 1);
		delete bookmarks_store.bookmarks[id];
		return;
	}
	console.error(`chrome.bookmarks.onRemoved: Bookmark #${id} is not found in parent #${info.parentId} at index ${info.index}.`);
});

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
