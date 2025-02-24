<script lang="ts">
	import {
		App,
		debounce,
		TFile,
		type Instruction,
		type SplitDirection,
	} from 'obsidian';
	import {
		CARD_VIEW_MODAL_HOTKEY_ACTION_IDS,
		HOTKEY_ACTION_INFO,
	} from 'Setting';
	import { createEventDispatcher, onDestroy, onMount } from 'svelte';
	import {writable} from 'svelte/store'
	import CardContainer from 'ui/CardContainer.svelte';
	import { app, plugin, switcher} from 'ui/store';
	import { convertHotkeyToText } from 'utils/Keymap';
	import {
		fuzzySearchInFilePaths,
		pickRandomly,
		searchInFiles,
		sortResultItemsInFilePathSearch,
		type FileSearchResultItem,
		search,
		getFoundAfterDelay,
		type SearchDetails
	} from 'utils/Search';
	import InfiniteScroll from 'svelte-infinite-scroll';
	import { getAPI } from "obsidian-dataview";

	// const
	const CARDS_PER_PAGE = 10;
	// const instructions: Instruction[] = [
	// 	{ command: '↑↓', purpose: 'to navigate' },
	// 	{ command: 'ctrl + n/p', purpose: 'to navigate' },
	// 	{ command: '↵', purpose: 'to open' },
	// 	{ command: 'ctrl + ↵', purpose: 'to open horizontally' },
	// 	{ command: 'ctrl + shift + ↵', purpose: 'to open vertically' },
	// 	{ command: 'esc', purpose: 'to dismiss' },
	// ];
	const instructions: (Instruction | undefined)[] =
		CARD_VIEW_MODAL_HOTKEY_ACTION_IDS.map((actionId) => {
			const hotkeys = $plugin.settings?.cardViewModalHotkeys[actionId];
			if (!hotkeys) return undefined;
			const purpose = 'to ' + HOTKEY_ACTION_INFO[actionId].short;
			const cmd: string = hotkeys
				.map((hotkey) => convertHotkeyToText(hotkey))
				.join(', ');
			return {
				command: cmd,
				purpose: purpose,
			};
		});
	instructions.push({
		command: 'esc',
		purpose: 'to dismiss',
	});

	// bind
	// let containerEl: HTMLElement | undefined | null;
	let inputEl: HTMLInputElement | undefined | null;
	let contentEl: HTMLElement | undefined | null;

	// state variables
	let results: FileSearchResultItem[];
	let selected = writable(0);
	let page = 0;
	let cards: CardContainer[] = [];
	// type SearchMode = 'normal' | 'recent';
	// let mode: SearchMode = 'recent';
	const context = new Map<string, number>();
	context.set("selectedIndex", selected)
	let lastFocusedIndex: number = 0;
	selected.subscribe(value => {
			console.log("lastFocusedIndex:" + lastFocusedIndex)
			if (value !== lastFocusedIndex) {
				if (cards[lastFocusedIndex] !== undefined) {
					cards[lastFocusedIndex].$set({ selected: false });
				}
			}
			lastFocusedIndex = value;
		});

	let lastItemId: number = 0;

	// debouncer
	const searchAndRenderDebouncer = debounce(searchAndRender, 100, true);

	// event dispatcher
	const dispatcher = createEventDispatcher();

	onMount(async () => {
		inputEl?.focus();

		results = await getResults('');
		if (contentEl instanceof HTMLElement) {
			renderCards(contentEl, results, 0);
		}
		focusOn(0);
	});

	onDestroy(() => {
		detachCards();
	});

	export function navigateForward() {
		// update selected
		let updated = true;
		$selected++;

		if ($selected >= results.length) {
			$selected = results.length - 1;
			updated = false;
		}

		focusOn($selected);

		// update page
		if (updated) {
			const shouldTransitNextPage = $selected % CARDS_PER_PAGE === 0;
			if (shouldTransitNextPage) {
				nextPage();
			}
		}
	}

	function nextPage() {
		page++;
		if (contentEl instanceof HTMLElement) {
			renderCards(contentEl, results, page);
		}
		focusOn($selected);
	}

	export function navigateBack() {
		// update selected
		let updated = true;
		$selected--;
		if ($selected < 0) {
			$selected = 0;
			updated = false;
		}

		focusOn($selected);

		// update page
		if (updated) {
			const shouldTransitPreviousPage =
				($selected + 1) % CARDS_PER_PAGE === 0;
			if (shouldTransitPreviousPage) {
				previousPage();
			}
		}
	}

	function previousPage() {
		page--;
		if (contentEl instanceof HTMLElement) {
			renderCards(contentEl, results, page);
		}
		focusOn($selected);
	}

	export async function open(direction?: SplitDirection) {
		if ($selected === undefined) {
			return;
		}
		const file = results[$selected]?.file;
		if (file === undefined) {
			return;
		}
		$switcher.shouldRestoreSelection = false;
		await openFile(file, direction);
		dispatcher('should-destroy');
	}

	export function selectedResult(): FileSearchResultItem | undefined {
		const result = results[$selected];
		return result;
	}

	async function onInput(evt: Event) {
		if (!(evt instanceof InputEvent)) return;
		const inputEl = evt.target;
		if (!(inputEl instanceof HTMLInputElement)) return;
		const changed = changeMode(inputEl, evt);
		if (changed) return;
		searchAndRenderDebouncer(inputEl);
	}

	async function searchAndRender(inputEl: HTMLInputElement) {
		// refresh
		$selected = 0;
		page = 0;
		detachCards();
		// contentEl?.empty(); // unnecessary. rather cause error when used

		results = await getResults(inputEl.value);
		if (contentEl instanceof HTMLElement) {
			renderCards(contentEl, results, 0);
		}
		focusOn(0);
	}

	function focusOn(id: number) {
		// const pos = id % CARDS_PER_PAGE; // id in results => position in cards
		const card = cards[id];
		if (!card) return;
		card.$set({ selected: true });
		lastFocusedIndex = id;
	}

	async function openFile(file: TFile, direction?: SplitDirection) {
		const leaf =
			direction !== undefined
				? $app.workspace.splitActiveLeaf(direction)
				: $app.workspace.getMostRecentLeaf();
		await leaf.openFile(file);
		$app.workspace.setActiveLeaf(leaf, true, true);
	}

	async function getResults(query: string): Promise<FileSearchResultItem[]> {
		let results: FileSearchResultItem[];
		const spaceTrimmedQuery = query.trimEnd();

		if (spaceTrimmedQuery === '') {
			const files = getRecentFiles($app);
			results = files.map((file) => {
				return { file: file, name: null, path: null, content: null };
			});
		} else if (spaceTrimmedQuery.startsWith("'")) {
			const trimmedQuery = spaceTrimmedQuery.replace(/^'/, '');
			const files = $app.vault.getFiles();
			const results = sortResultItemsInFilePathSearch(
				fuzzySearchInFilePaths(trimmedQuery, files)
			);
			return results;
		} else if (spaceTrimmedQuery.startsWith(":")) {
			const trimmedQuery = spaceTrimmedQuery.replace(/^:/, '');
			search(trimmedQuery);
			await getFoundAfterDelay();
			const searchResults = await getFoundAfterDelay()
			const files = Array.from(searchResults.keys())
			console.log(files);
			const results = files.map((k) => {
				return {
					file: searchResults.get(k).file,
					name: k.basename,
					path: k.path,
					content: searchResults.get(k).content
				}
			})
			return results;
		} else if (spaceTrimmedQuery.startsWith("~")) {
			const trimmedQuery = spaceTrimmedQuery.replace(/^~/, '');
			if (window.dvSearchResults === undefined) {
				return [];
			}
			const files = window.dvSearchResults.map(r => {
				return $app.vault.getAbstractFileByPath(r.file.path)
			});
			return files.map(f => {
				return {
					file: f,
					name: f.basename,
					path: f.path,
					content: undefined
				}
			})
			// search(trimmedQuery);
			// await getFoundAfterDelay();
			// const searchResults = await getFoundAfterDelay()
			// const files = Array.from(searchResults.keys())
			// console.log(files);
			// const results = files.map((k) => {
			// 	return {
			// 		file: searchResults.get(k).file,
			// 		name: k.basename,
			// 		path: k.path,
			// 		content: searchResults.get(k).content
			// 	}
			// })
			// return results;
		} else if (spaceTrimmedQuery.startsWith(';')) {
			const trimmedQuery = spaceTrimmedQuery.replace(/^;/, '');
			const files = $app.vault.getFiles();
			const searchResults = fuzzySearchInFilePaths(trimmedQuery, files);
			const results = pickRandomly(
				searchResults,
				Math.min(CARDS_PER_PAGE, searchResults.length)
			);
			return results;
		} else {
			const files = getRecentFiles($app);
			const results = await searchInFiles($app, spaceTrimmedQuery, files);
			return results;
		}
		return results;
	}

	/**
	 *
	 * @returns whether mode change occurs
	 */
	function changeMode(inputEl: HTMLInputElement, evt: InputEvent): boolean {
		if (evt.data === ' ' && inputEl.value === evt.data) {
			evt.preventDefault();
			inputEl.value = "'";
			return true;
		}
		return false;
	}

	function renderCards(
		contentEl: HTMLElement,
		results: FileSearchResultItem[],
		page: number
	) {
		// refresh
		// contentEl.empty(); // unnecessary. rather cause error when used
		// detachCards();

		for (
			let id = CARDS_PER_PAGE * page;
			id < CARDS_PER_PAGE * (page + 1);
			id++
		) {
			const result = results[id];
			if (!result) continue;
			const card = new CardContainer({
				target: contentEl,
				props: {
					id: id,
					file: result.file,
					matches: result?.path?.matches ?? [],
					selected: false,
					focusEl: inputEl
				},
				context: context
			});
			card.$on('click', () => {
				$selected = id;
				open();
			});
			cards.push(card);
		}
	}

	function detachCards() {
		cards.forEach((card) => card.$destroy());
		cards = [];
	}

	function getRecentFiles(app: App): TFile[] {
		const files = app.workspace
			.getLastOpenFiles()
			.map((path) => app.vault.getAbstractFileByPath(path))
			.filter((file) => file instanceof TFile) as TFile[];
		return files;
	}
</script>

<div class="modal">
	<div
		class="modal-background"
		on:click={() => {
			dispatcher('should-destroy');
		}}
	/>

	<div class="prompt-container">
		<input
			class="prompt-input"
			placeholder="Hit space key to toggle the normal search mode"
			bind:this={inputEl}
			on:input={onInput}
		/>
		<div class="prompt-instruction-container">
			{#each instructions as instruction}
				{#if instruction}
					<div class="prompt-instruction">
						<span class="prompt-instruction-command"
							>{instruction.command}</span
						>
						<span>{instruction.purpose}</span>
					</div>
				{/if}
			{/each}
		</div>
	</div>

	<div class="cards-container" bind:this={contentEl} >
		<!-- <div>{$selected}</div> -->
		<InfiniteScroll threshold={100} on:loadMore={() => {
			console.log(console.log("hello")); 
			page++;
			if (contentEl instanceof HTMLElement) {
				renderCards(contentEl, results, page);
			}

		}} />
	</div>
</div>

<style>
	.modal {
		display: flex;
		align-items: center;
		flex-direction: column;
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		z-index: var(--layer-modal);

		padding: 30px 50px;

		/* reset the default obsidian style */
		max-width: unset;
		border-radius: unset;
		border: unset;
		background-color: unset;
	}

	.modal-background {
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		background-color: var(--background-modifier-cover);
	}

	.prompt-container {
		margin-bottom: 20px;
		background-color: var(--background-primary);
		padding: 10px;
		border-radius: 8px;
		display: flex;
		flex-direction: column;
		align-items: center;
		width: fit-content;
		z-index: 1; /* to put this in front of background */
	}

	.prompt-input {
		max-width: 700px;
		padding: 9px 20px 8px 20px;
		font-size: 16px;
		border-radius: 6px;
		height: 40px;

		background: var(--background-modifier-form-field);
		border: 1px solid var(--background-modifier-border);
		color: var(--text-normal);
	}

	.prompt-input:focus,
	.prompt-input:active {
		border-color: var(--interactive-accent);
	}

	.prompt-instruction-container {
		display: unset;
		margin-right: 0;
		padding: 10px 10px 2px 10px;
		display: flex;
		flex-direction: row;
		flex-wrap: wrap;
		justify-content: center;
	}

	.prompt-instruction-command {
		font-size: 12px;
		font-weight: 900;
		margin-right: 4px;
	}

	.cards-container {
		display: grid;
		grid-template-columns: repeat(5, minmax(0, 1fr));
		/* grid-template-rows: repeat(attr(data-row), 1fr); */
		grid-template-rows: repeat(2, 800px);
		grid-gap: 20px;
		height: 100%;
		width: 100%;
		min-height: 0;
		z-index: 1; /* to put this in front of background */
		grid-auto-rows: 800px;
		overflow: scroll;
	}
</style>
