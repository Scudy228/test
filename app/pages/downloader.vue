<template>
	<LayoutTile
		title="YouTube Downloader"
		description="Glissez-déposez un lien YouTube pour le télécharger en HD dans votre dossier Téléchargements."
	>
		<div class="space-y-5">
			<div
				class="border-2 border-dashed rounded-xl p-10 text-center transition-all select-none"
				:class="isDragging
					? 'border-primary bg-primary/10'
					: 'border-(--ui-border) hover:border-primary/60 cursor-pointer'"
				@dragover.prevent="isDragging = true"
				@dragleave.prevent="isDragging = false"
				@drop.prevent="onDrop"
				@click="urlInputRef?.input?.focus()"
			>
				<div class="flex flex-col items-center gap-3">
					<UIcon name="lucide:youtube" class="size-12 text-red-500" />
					<p class="text-base font-semibold">
						Glissez un lien YouTube ici
					</p>
					<p class="text-sm text-(--ui-text-muted)">
						ou collez-le dans le champ ci-dessous
					</p>
				</div>
			</div>

			<UFormField label="Lien YouTube" name="url">
				<UInput
					ref="urlInputRef"
					v-model="url"
					placeholder="https://www.youtube.com/watch?v=..."
					size="lg"
					:leading-icon="isValidUrl ? 'lucide:check-circle' : 'lucide:link'"
					:color="url && !isValidUrl ? 'error' : undefined"
					:ui="{ leadingIcon: isValidUrl ? 'text-green-500' : undefined }"
				/>
			</UFormField>

			<UFormField label="Qualité vidéo" name="format">
				<USelect v-model="selectedFormat" :items="formats" size="lg" value-key="value" label-key="label" />
			</UFormField>

			<UFormField label="Navigateur pour les cookies" name="browser">
				<USelect v-model="selectedBrowser" :items="browsers" size="lg" value-key="value" label-key="label" />
			</UFormField>

			<UFormField
				v-if="selectedBrowser === '__file__'"
				label="Chemin vers le fichier cookies.txt"
				name="cookiesFile"
				description="Exporte tes cookies YouTube depuis Chrome avec l'extension « Get cookies.txt LOCALLY »"
			>
				<UInput
					v-model="cookiesFile"
					placeholder="/Users/toi/Downloads/cookies.txt"
					size="lg"
					leading-icon="lucide:file"
				/>
			</UFormField>

			<UCheckbox v-model="noPlaylist" label="Vidéo seule (ignorer la playlist)" />

			<UButton
				size="lg"
				block
				:disabled="!isValidUrl || isDownloading"
				:loading="isDownloading"
				@click="download"
			>
				{{ isDownloading ? 'Téléchargement en cours…' : 'Télécharger' }}
			</UButton>

			<UAlert
				v-if="successMsg"
				color="success"
				variant="soft"
				icon="lucide:check-circle"
				:title="successMsg"
			/>

			<UAlert
				v-if="errorMsg"
				color="error"
				variant="soft"
				icon="lucide:alert-circle"
				:title="errorMsg"
			/>

			<div v-if="output" class="space-y-2">
				<p class="text-sm font-medium text-(--ui-text-muted)">
					Journal de téléchargement
				</p>
				<UTextarea
					:model-value="output"
					readonly
					:rows="10"
					class="font-mono text-xs"
				/>
			</div>
		</div>
	</LayoutTile>
</template>

<script lang="ts" setup>
	definePageMeta({
		name: "YouTube Downloader",
		icon: "lucide:download",
		description: "Télécharger des vidéos YouTube en HD",
		category: "other"
	});

	const formats: { label: string; value: string }[] = [
		{ label: "Meilleure qualité (4K/2K/1080p)", value: "bestvideo[ext=mp4]+bestaudio[ext=m4a]/bestvideo+bestaudio/best" },
		{ label: "1080p HD", value: "bestvideo[height<=1080][ext=mp4]+bestaudio[ext=m4a]/bestvideo[height<=1080]+bestaudio/best[height<=1080]" },
		{ label: "720p HD", value: "bestvideo[height<=720][ext=mp4]+bestaudio[ext=m4a]/bestvideo[height<=720]+bestaudio/best[height<=720]" },
		{ label: "480p", value: "bestvideo[height<=480][ext=mp4]+bestaudio[ext=m4a]/bestvideo[height<=480]+bestaudio/best[height<=480]" },
		{ label: "Audio seulement (MP3)", value: "bestaudio[ext=m4a]/bestaudio" }
	];

	const browsers: { label: string; value: string }[] = [
		{ label: "Chrome", value: "chrome" },
		{ label: "Firefox", value: "firefox" },
		{ label: "Brave", value: "brave" },
		{ label: "Fichier cookies.txt…", value: "__file__" },
		{ label: "Aucun", value: "" }
	];

	const FFMPEG_LOCATIONS: Record<string, string> = {
		macos: "/opt/homebrew/bin",
		linux: "/usr/bin",
		windows: "C:\\ffmpeg\\bin"
	};

	const urlInputRef = ref<any>(null);
	const url = ref("");
	const selectedFormat = ref<string>(formats[0]!.value);
	const selectedBrowser = ref<string>("chrome");
	const cookiesFile = ref("");
	const noPlaylist = ref(true);
	const isDragging = ref(false);
	const isDownloading = ref(false);
	const output = ref("");
	const successMsg = ref("");
	const errorMsg = ref("");

	const YOUTUBE_REGEX = /^(https?:\/\/)?(www\.)?(youtube\.com\/(watch\?.*v=|shorts\/|playlist\?)|youtu\.be\/)[\w\-&=?%]+/;

	const isValidUrl = computed(() => YOUTUBE_REGEX.test(url.value.trim()));

	const onDrop = (e: DragEvent) => {
		isDragging.value = false;
		const dropped = e.dataTransfer?.getData("text/uri-list") || e.dataTransfer?.getData("text/plain") || "";
		if (dropped) {
			url.value = dropped.trim().split("\n")[0]!;
		}
	};

	const download = async () => {
		if (!isValidUrl.value || isDownloading.value) return;

		const platform = await useTauriOsPlatform();
		const isAudioOnly = selectedFormat.value.startsWith("bestaudio");
		const outputTemplate = "~/Downloads/%(title)s.%(ext)s";
		const ffmpegLocation = FFMPEG_LOCATIONS[platform] ?? "";

		const args: string[] = [];

		if (ffmpegLocation) args.push("--ffmpeg-location", ffmpegLocation);
		if (selectedBrowser.value === "__file__" && cookiesFile.value.trim()) {
			args.push("--cookies", cookiesFile.value.trim());
		} else if (selectedBrowser.value && selectedBrowser.value !== "__file__") {
			args.push("--cookies-from-browser", selectedBrowser.value);
		}
		if (noPlaylist.value) args.push("--no-playlist");

		args.push("-f", selectedFormat.value);

		if (isAudioOnly) {
			args.push("--extract-audio", "--audio-format", "mp3");
		} else {
			args.push("--merge-output-format", "mp4");
		}

		args.push("-o", outputTemplate, url.value.trim());

		isDownloading.value = true;
		output.value = "";
		successMsg.value = "";
		errorMsg.value = "";

		try {
			const command = useTauriShellCommand.sidecar("binaries/yt-dlp", args);

			command.stdout.on("data", (line: string) => {
				output.value += line + "\n";
			});
			command.stderr.on("data", (line: string) => {
				output.value += line + "\n";
			});

			const { code } = await new Promise<{ code: number | null }>((resolve, reject) => {
				command.on("close", resolve);
				command.on("error", reject);
				command.spawn().catch(reject);
			});

			if (code === 0) {
				successMsg.value = "Téléchargement terminé ! Fichier enregistré dans Téléchargements.";
				url.value = "";
			} else {
				errorMsg.value = "Le téléchargement a échoué. Consultez le journal ci-dessous.";
			}
		} catch (err: any) {
			errorMsg.value = `Impossible de lancer yt-dlp : ${err?.message ?? err}`;
			output.value += String(err);
		} finally {
			isDownloading.value = false;
		}
	};
</script>
