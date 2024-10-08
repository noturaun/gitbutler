<script lang="ts">
	// import { Project } from '$lib/backend/projects';
	import ActionView from '$lib/layout/ActionView.svelte';
	import { ModeService, type EditModeMetadata } from '$lib/modes/service';
	import { UncommitedFilesWatcher } from '$lib/uncommitedFiles/watcher';
	import { getContext } from '$lib/utils/context';
	import Button from '@gitbutler/ui/Button.svelte';
	import InfoButton from '@gitbutler/ui/InfoButton.svelte';
	import Avatar from '@gitbutler/ui/avatar/Avatar.svelte';
	import FileListItem from '@gitbutler/ui/file/FileListItem.svelte';
	import type { RemoteFile } from '$lib/vbranches/types';
	import type { FileStatus } from '@gitbutler/ui/file/types';

	interface Props {
		editModeMetadata: EditModeMetadata;
	}

	const { editModeMetadata }: Props = $props();

	// const project = getContext(Project);
	const uncommitedFileWatcher = getContext(UncommitedFilesWatcher);
	const modeService = getContext(ModeService);

	const uncommitedFiles = uncommitedFileWatcher.uncommitedFiles;

	let modeServiceAborting = $state<'inert' | 'loading' | 'completed'>('inert');
	let modeServiceSaving = $state<'inert' | 'loading' | 'completed'>('inert');

	let initialFiles = $state<RemoteFile[]>([]);

	$effect(() => {
		modeService.getInitialIndexState().then((files) => {
			initialFiles = files;
		});
	});

	interface FileEntry {
		name: string;
		path: string;
		conflicted: boolean;
		status?: FileStatus;
	}

	const files = $derived.by(() => {
		const initialFileMap = new Map<string, RemoteFile>();
		const uncommitedFileMap = new Map<string, RemoteFile>();
		const outputMap = new Map<string, FileEntry>();

		// Build maps of files
		{
			initialFiles.forEach((initialFile) => {
				initialFileMap.set(initialFile.path, initialFile);
			});

			$uncommitedFiles.forEach(([uncommitedFile]) => {
				uncommitedFileMap.set(uncommitedFile.path, uncommitedFile);
			});
		}

		// Create output
		{
			initialFiles.forEach((initialFile) => {
				const isDeleted = uncommitedFileMap.has(initialFile.path);

				outputMap.set(initialFile.path, {
					name: initialFile.filename,
					path: initialFile.path,
					conflicted: initialFile.looksConflicted,
					status: isDeleted ? undefined : 'D'
				});
			});

			$uncommitedFiles.forEach(([uncommitedFile]) => {
				const initialFile = initialFileMap.get(uncommitedFile.path);
				if (initialFile) {
					const fileChanged = initialFile.hunks.some(
						(hunk) => !uncommitedFile.hunks.map((hunk) => hunk.diff).includes(hunk.diff)
					);

					if (fileChanged && !uncommitedFile.looksConflicted) {
						// All initial entries should have been added to the map,
						// so we can safly assert that it will be present
						const outputFile = outputMap.get(uncommitedFile.path)!;
						outputFile.status = 'M';
						outputFile.conflicted = false;
					}
				} else {
					outputMap.set(uncommitedFile.path, {
						name: uncommitedFile.filename,
						path: uncommitedFile.path,
						conflicted: false,
						status: 'A'
					});
				}
			});
		}

		const files = Array.from(outputMap.values());
		files.sort((a, b) => a.path.localeCompare(b.path));

		return files;
	});

	async function abort() {
		modeServiceAborting = 'loading';

		await modeService.abortEditAndReturnToWorkspace();

		modeServiceAborting = 'completed';
	}

	async function save() {
		modeServiceSaving = 'loading';

		await modeService.saveEditAndReturnToWorkspace();

		modeServiceSaving = 'completed';
	}
</script>

<ActionView
	paddings={{
		left: 48
	}}
>
	<h2 class="editmode__title text-18 text-body text-bold">
		You are editing commit <span class="code-string">
			{editModeMetadata.commitOid.slice(0, 7)}
		</span>
		<InfoButton title="Edit Mode">
			Edit Mode lets you modify an existing commit in isolation or resolve conflicts. Any changes
			made, including new files, will be added to the selected commit.
		</InfoButton>
	</h2>

	<div class="commit-group">
		<div class="commit-line__container">
			<div class="commit-line__top-line"></div>
			<div class="commit-line__avatar">
				<Avatar srcUrl="oops" tooltip="author" />
			</div>
			<div class="commit-line__bottom-line"></div>
		</div>

		<div class="commit-data">
			<div class="card commit-card">
				<h3 class="text-13 text-semibold commit-card__title">Awesome title</h3>
				<div class="text-11 commit-card__details">
					<span class="">{editModeMetadata.commitOid.slice(0, 7)}</span>
					<span class="commit-card__divider">•</span>
					<span class="">Author</span>
				</div>

				<div class="commit-card__type-indicator"></div>
			</div>

			<div class="card files">
				<h3 class="text-13 text-semibold header">Commit files</h3>
				{#each files as file}
					<div class="file">
						<FileListItem
							fileName={file.name}
							filePath={file.path}
							fileStatus={file.status}
							conflicted={file.conflicted}
							fileStatusStyle={file.status === 'M' ? 'full' : 'dot'}
							clickable={false}
						/>
					</div>
				{/each}
			</div>
		</div>
	</div>

	<!-- <div class="editmode__helpbox"> -->
	<p class="text-12 text-body editmode__helptext">
		Please don't make any commits while in edit mode.
		<br />
		To exit edit mode, use the provided actions.
	</p>
	<!-- </div> -->

	<div class="editmode__actions">
		<Button style="ghost" outline onclick={abort} disabled={modeServiceAborting === 'loading'}>
			Cancel
		</Button>
		<Button
			style="pop"
			kind="solid"
			icon="tick-small"
			onclick={save}
			disabled={modeServiceSaving === 'loading'}
		>
			Save and exit
		</Button>
	</div>
</ActionView>

<style lang="postcss">
	.editmode__title {
		color: var(--clr-text-1);
		margin-bottom: 12px;
	}

	.editmode__actions {
		display: flex;
		gap: 8px;
		padding-bottom: 24px;
		flex-wrap: wrap;
		justify-content: flex-end;
	}

	.files {
		margin-bottom: 12px;
		overflow: hidden;
		padding-bottom: 8px;

		& .header {
			margin-left: 16px;
			margin-top: 16px;
			margin-bottom: 8px;
		}

		& .file {
			border-bottom: 1px solid var(--clr-border-3);
			&:last-child {
				border-bottom: none;
			}
		}
	}

	.code-string {
		margin-right: 2px;
	}

	/* COMMIT */
	.commit-group {
		position: relative;
		display: flex;
		gap: 14px;
	}

	.commit-data {
		position: relative;
		display: flex;
		flex-direction: column;
		gap: 4px;
		width: 100%;
	}

	/* COMMIT CARD */
	.commit-card {
		position: relative;
		padding: 14px 14px 14px 16px;
		gap: 6px;
		overflow: hidden;
	}

	.commit-card__title {
		color: var(--clr-text-1);
	}

	.commit-card__details {
		display: flex;
		gap: 4px;
		color: var(--clr-text-2);
	}

	.commit-card__type-indicator {
		position: absolute;
		top: 0;
		left: 0;
		width: 4px;
		height: 100%;
		background-color: var(--clr-commit-local);
	}

	/* COMMIT LINE */
	.commit-line__container {
		position: absolute;
		top: 0;
		left: -26px;
		display: flex;
		flex-direction: column;
		align-items: center;
		height: 100%;
	}

	.commit-line__avatar {
		position: absolute;
		top: 15px;
		left: 50%;
		transform: translateX(-50%);
		border: 2px solid var(--clr-commit-local);
		border-radius: 50%;
	}

	.commit-line__top-line {
		width: 2px;
		height: 48px;
		margin-top: -26px;
		background: linear-gradient(180deg, transparent 0%, var(--clr-commit-local) 100%);
	}

	.commit-line__bottom-line {
		width: 2px;
		height: 100%;
		background: linear-gradient(180deg, var(--clr-commit-local) 0%, transparent 100%);
	}

	/* .editmode__helpbox {
		color: var(--clr-text-2);
		margin-bottom: 16px;
		padding: 12px;
		border-radius: var(--radius-m);
		background-color: var(--clr-bg-1-muted);
	} */

	.editmode__helptext {
		color: var(--clr-text-3);
		margin-bottom: 16px;
	}
</style>
