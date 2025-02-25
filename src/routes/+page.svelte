<script lang="ts">
	import "../app.postcss";

	import { dev, browser } from "$app/environment";
	import { afterUpdate, onMount, tick, type ComponentProps } from "svelte";

	import { CopyButton, Editor } from "drab";

	import { process } from "robino/util/md";

	import { inject } from "@vercel/analytics";

	import gettingStarted from "$lib/gettingStarted.md?raw";

	import FormatButton from "$lib/components/FormatButton.svelte";
	import Metrics from "$lib/components/Metrics.svelte";
	import PrintButton from "$lib/components/PrintButton.svelte";
	import Slides from "$lib/components/Slides.svelte";

	import { codeEval } from "$lib/utilities/codeEval";
	import { formatMd } from "$lib/utilities/formatMd";

	// svg
	import Bullet from "$lib/components/svg/Bullet.svelte";
	import Blockquote from "$lib/components/svg/Blockquote.svelte";
	import Anchor from "$lib/components/svg/Anchor.svelte";
	import Image from "$lib/components/svg/Image.svelte";
	import Table from "$lib/components/svg/Table.svelte";
	import Code from "$lib/components/svg/Code.svelte";
	import View from "$lib/components/svg/View.svelte";
	import Edit from "$lib/components/svg/Edit.svelte";
	import Document from "$lib/components/svg/Document.svelte";
	import Slideshow from "$lib/components/svg/Slideshow.svelte";
	import ZoomOut from "$lib/components/svg/ZoomOut.svelte";
	import ZoomIn from "$lib/components/svg/ZoomIn.svelte";
	import Open from "$lib/components/svg/Open.svelte";
	import Save from "$lib/components/svg/Save.svelte";
	import Copy from "$lib/components/svg/Copy.svelte";
	import CopyComplete from "$lib/components/svg/CopyComplete.svelte";
	import CodeBracket from "$lib/components/svg/CodeBracket.svelte";

	inject({ mode: dev ? "development" : "production" });

	/** raw text that the user enters into the `textarea` element */
	let content = "";

	/** controls the expansion of the preview area */
	let viewMode = false;

	/** selection start of the `textArea` inside of `Editor` */
	let selectionStartTextarea: number;

	let currentSlide: number;

	let placeholder = "";

	let file: File | null;
	let fileHandle: FileSystemFileHandle | null;

	/** `true` if the browser supports the `window.showOpenFilePicker` method */
	let supported = false;
	if (browser) supported = Boolean(window.showOpenFilePicker);

	const fontSizes = [
		"prose-sm",
		"prose-base",
		"prose-lg",
		"prose-xl",
		"prose-2xl",
	];

	const fontFamilies = [
		"font-sans",
		"font-round",
		"font-serif",
		"font-antique",
		"font-mono",
	];

	const colors = {
		prose: ["", "prose-teal", "prose-sky", "prose-rose"],
		medium: ["bg-gray-500", "bg-teal-500", "bg-sky-500", "bg-rose-500"],
		dark: ["bg-gray-900", "bg-teal-950", "bg-sky-950", "bg-gray-800"],
	};

	const viewTypes = ["document", "slideshow"] as const;

	interface Preferences {
		fontSize: number;
		fontFamily: number;
		color: number;
		viewType: (typeof viewTypes)[number];
	}

	let preferences: Preferences = {
		fontSize: 1,
		fontFamily: 0,
		color: 0,
		viewType: "document",
	};

	const savePreferences = () => {
		localStorage.setItem("preferences", JSON.stringify(preferences));
	};

	/** passed in as a prop for the `Editor.svelte` controls */
	const contentElements: ComponentProps<Editor>["contentElements"] = [
		{
			title: "Heading",
			text: "# ",
			display: "block",
			icon: "H",
		},
		{
			title: "Bullet",
			text: "- ",
			display: "block",
			icon: Bullet,
		},
		{
			title: "Blockquote",
			text: "> ",
			display: "block",
			icon: Blockquote,
		},
		{
			title: "Italic",
			text: "*",
			display: "wrap",
			icon: "I",
			class: "italic",
		},
		{
			title: "Bold",
			text: "**",
			display: "wrap",
			icon: "B",
		},
		{
			title: "Strikethrough",
			text: "~",
			display: "wrap",
			icon: "S",
			class: "line-through font-normal",
		},
		{
			title: "Anchor",
			text: "[text](href)",
			display: "inline",
			icon: Anchor,
			key: "[",
		},
		{
			title: "Image",
			text: "![alt](src)",
			display: "inline",
			icon: Image,
			key: "]",
		},
		{
			title: "Table",
			text: "| th  | th  |\n| --- | --- |\n| td  | td  |\n| td  | td  |",
			display: "inline",
			icon: Table,
			key: "\\",
		},
		{
			title: "Code",
			text: "`",
			display: "wrap",
			icon: CodeBracket,
		},
		{
			title: "Slide",
			text: "---",
			display: "inline",
			icon: Slideshow,
		},
	];

	const options: FilePickerOptions = {
		types: [
			{
				description: "markdown",
				accept: {
					"text/markdown": [".md", ".mdx", ".mdoc", ".markdoc", ".svx"],
				},
			},
		],
		excludeAcceptAllOption: true,
	};

	const open = async () => {
		[fileHandle] = await window.showOpenFilePicker(options);
		file = await fileHandle.getFile();
		content = await file.text();
		toggleView();
	};

	const saveAs = async () => {
		const handle = await window.showSaveFilePicker(options);
		const writable = await handle.createWritable();
		await writable.write(content);
		await writable.close();
		file = await handle.getFile();
		fileHandle = handle;
	};

	const save = async () => {
		if (fileHandle && !viewMode) {
			const writable = await fileHandle.createWritable();
			await writable.write(content);
			await writable.close();
		}
	};

	const dropFile = async (e: DragEvent) => {
		const items = e.dataTransfer?.items;
		if (items && items[0]) {
			const item = items[0];
			if (item.kind === "file") {
				// @ts-ignore - not supported by all browsers
				if (item.getAsFileSystemHandle) {
					e.preventDefault();
					const handle = await item.getAsFileSystemHandle();
					// since `item.kind === "file"` it will be a `FileSystemFileHandle`
					fileHandle = handle as FileSystemFileHandle;
					file = await fileHandle.getFile();
					content = await file.text();
				}
			}
		}
	};

	const toggleView = () => {
		const toggleView = () => {
			viewMode = !viewMode;
		};

		// @ts-expect-error - not supported by all browsers
		if (document.startViewTransition) {
			// @ts-expect-error - not supported by all browsers
			document.startViewTransition(() => {
				toggleView();
			});
		} else {
			toggleView();
		}
	};

	const changeViewType = (type: typeof preferences.viewType) => {
		preferences.viewType = type;
		savePreferences();
	};

	const changeProseSize = (action: "increase" | "decrease") => {
		if (action === "increase") {
			if (preferences.fontSize < fontSizes.length - 1) preferences.fontSize++;
		} else {
			if (preferences.fontSize > 0) preferences.fontSize--;
		}
		savePreferences();
	};

	const changeColor = () => {
		if (preferences.color < colors.prose.length - 1) {
			preferences.color++;
		} else {
			preferences.color = 0;
		}
		savePreferences();
	};

	const changeFontFamily = () => {
		if (preferences.fontFamily < fontFamilies.length - 1) {
			preferences.fontFamily++;
		} else {
			preferences.fontFamily = 0;
		}
		savePreferences();
	};

	const onKeyUp = (e: KeyboardEvent) => {
		save();
		if (e.key === "i") {
			document.querySelector("textarea")?.focus();
		}
		if (e.key === "Escape") {
			toggleView();
		} else {
			findCurrentSlide();
		}
	};

	const onKeyDown = async (e: KeyboardEvent) => {
		if ((e.ctrlKey || e.metaKey) && e.key === "s") {
			e.preventDefault();
			content = await formatMd(content);
		}
	};

	const findCurrentSlide = () => {
		if (preferences.viewType === "slideshow" && !viewMode) {
			const s = content.slice(0, selectionStartTextarea);
			let curr = s.split("\n\n---\n").length - 1;
			if (s.startsWith("---\n")) curr++; // if first line === `---`, considered an <hr>
			currentSlide = curr;
		}
	};

	const setPlaceholder = () => {
		placeholder = gettingStarted.trim();
		contentElements.forEach((el) => {
			if (el.key)
				placeholder += `\n| ${el.title} | <kbd>CTRL</kbd> + <kbd>${el.key} </kbd> |`;
		});
	};

	setPlaceholder();

	onMount(() => {
		const saved = localStorage.getItem("preferences");
		if (saved) {
			preferences = JSON.parse(saved);
		} else {
			savePreferences();
		}
	});

	afterUpdate(async () => {
		await tick();
		codeEval();
	});

	$: html = process(content ? content : placeholder).html;
</script>

<svelte:document on:keyup={onKeyUp} on:keydown={onKeyDown} />

<div
	class="flex h-[100dvh] flex-col bg-gray-950 text-gray-50 selection:bg-gray-400 selection:bg-opacity-40"
	on:drop={dropFile}
	role="main"
>
	{#if !viewMode}
		<header class="flex justify-between p-3 text-sm">
			<nav class="flex w-full items-center justify-between sm:w-fit">
				<div class="flex">
					{#if supported}
						<button title="Open (or drag and drop)" class="btn" on:click={open}>
							<Open />
							<span class="hidden lg:inline">Open</span>
						</button>
						<button title="Save As" class="btn" on:click={saveAs}>
							<Save />
							<span class="hidden lg:inline">Save As</span>
						</button>
					{:else}
						<a
							href="data:text/plain,{content}"
							download="Untitled.md"
							title="Download"
							class="btn"
						>
							<Save />
							<span class="hidden lg:inline">Download</span>
						</a>
					{/if}
					<CopyButton class="btn" blobParts={[content]}>
						<Copy />
						<span class="hidden lg:inline">Copy</span>
						<span
							class="flex items-center justify-center gap-1"
							slot="complete"
						>
							<CopyComplete />
							<span class="hidden lg:inline">Copy</span>
						</span>
					</CopyButton>
					<CopyButton class="btn" title="Copy HTML" blobParts={[html]}>
						<Code />
						<span class="hidden lg:inline">Copy HTML</span>
						<span
							class="flex items-center justify-center gap-1"
							slot="complete"
						>
							<CopyComplete />
							<span class="hidden lg:inline">Copy HTML</span>
						</span>
					</CopyButton>
					<PrintButton innerHtml={html} />
					<FormatButton bind:text={content} />
					<button title="View" class="btn lg:hidden" on:click={toggleView}>
						<View />
					</button>
				</div>
			</nav>
			<div class="hidden px-4 py-2 font-bold sm:block">
				{file?.name ? file.name : "Typo"}
			</div>
		</header>
	{/if}
	<main class="grid grow overflow-hidden {viewMode ? '' : 'lg:grid-cols-2'}">
		{#if !viewMode}
			<div class="flex h-full flex-col">
				<Editor
					classTextarea="grow resize-none appearance-none overflow-y-auto p-6 font-mono text-sm transition placeholder:text-gray-400 focus:outline-none {colors
						.dark[preferences.color]}"
					classControls="flex flex-wrap p-3"
					classButton="btn"
					{contentElements}
					placeholderTextarea="# Title"
					bind:valueTextarea={content}
					bind:selectionStartTextarea
				/>
			</div>
		{/if}
		<div
			style="view-transition-name: preview;"
			class="flex-col lg:flex {viewMode ? 'flex' : 'hidden'}"
		>
			<div
				class="grow overflow-y-auto border-gray-100 bg-white dark:border-y dark:border-gray-900 dark:bg-gray-950 {viewMode
					? 'max-h-[calc(100dvh-3.75rem)]'
					: 'max-h-[calc(100dvh-7.5rem)]'}"
				class:border-y={preferences.viewType === "document" && viewMode}
				class:dark:border-none={preferences.viewType === "slideshow" &&
					viewMode}
			>
				<!-- content -->
				<div
					class="prose dark:prose-invert prose-pre:bg-gray-900 prose-img:rounded-lg mx-auto h-full max-w-[72ch] break-words {fontSizes[
						preferences.fontSize
					]} {colors.prose[preferences.color]} {fontFamilies[
						preferences.fontFamily
					]}"
				>
					{#if preferences.viewType === "document"}
						<div class="p-8">
							{@html html}
						</div>
					{:else if preferences.viewType === "slideshow"}
						<Slides bind:viewMode bind:currentSlide {html} />
					{/if}
				</div>
			</div>
			<div
				class="group flex justify-between p-3"
				class:bg-white={viewMode}
				class:text-gray-950={viewMode}
				class:dark:bg-gray-950={viewMode}
				class:dark:text-gray-50={viewMode}
			>
				<!-- viewType controls -->
				<div class="flex">
					{#each viewTypes as type}
						<button
							class="btn group-hover:opacity-100"
							class:opacity-0={viewMode}
							disabled={preferences.viewType === type}
							on:click={() => changeViewType(type)}
							title={type}
						>
							{#if type === "document"}
								<Document />
							{:else if type === "slideshow"}
								<Slideshow />
							{/if}
						</button>
					{/each}
					<div
						class="transition group-hover:opacity-100"
						class:opacity-0={viewMode}
					>
						<Metrics {content} />
					</div>
				</div>
				<div class="flex">
					<button
						title="Change Color"
						class="btn group-hover:opacity-100"
						class:opacity-0={viewMode}
						on:click={changeColor}
					>
						<div
							class="h-5 w-5 rounded-full border-2 border-gray-50 {colors
								.medium[preferences.color]}"
							class:border-gray-950={viewMode}
							class:dark:border-gray-50={viewMode}
						/>
					</button>
					<button
						title="Change Font"
						class="btn group-hover:opacity-100 {fontFamilies[
							preferences.fontFamily
						]}"
						class:opacity-0={viewMode}
						on:click={changeFontFamily}
						aria-label={preferences.fontFamily ? "sans-serif" : "serif"}
					>
						F
					</button>
					<button
						title="Decrease Font Size"
						class="btn group-hover:opacity-100"
						class:opacity-0={viewMode}
						disabled={preferences.fontSize < 1}
						on:click={() => changeProseSize("decrease")}
					>
						<ZoomOut />
					</button>
					<button
						title="Increase Font Size"
						class="btn group-hover:opacity-100"
						class:opacity-0={viewMode}
						disabled={preferences.fontSize >= fontSizes.length - 1}
						on:click={() => changeProseSize("increase")}
					>
						<ZoomIn />
					</button>
					<!-- viewMode toggle -->
					<button
						title={viewMode ? "Edit" : "View"}
						class="btn"
						on:click={toggleView}
					>
						{#if viewMode}
							<Edit />
						{:else}
							<View />
						{/if}
					</button>
				</div>
			</div>
		</div>
	</main>
</div>
