<script lang="ts">
	import Markdown from '$lib/components/Markdown.svelte';
	import { dismissToast, toastStore } from '$lib/notifications/toasts';
	import InfoMessage from '$lib/shared/InfoMessage.svelte';
	import { slide } from 'svelte/transition';
</script>

<div class="toast-controller hide-native-scrollbar">
	<!-- eslint-disable-next-line svelte/valid-compile -->
	{#each $toastStore as toast (toast.id)}
		<div transition:slide={{ duration: 170 }}>
			<InfoMessage
				style={toast.style ?? 'neutral'}
				secondary="Dismiss"
				error={toast.error}
				on:secondary={() => dismissToast(toast.id)}
				shadow
			>
				<svelte:fragment slot="title">
					{toast.title}
				</svelte:fragment>

				<svelte:fragment slot="content">
					{#if toast.message}
						<Markdown content={toast.message} />
					{/if}
				</svelte:fragment>
			</InfoMessage>
		</div>
	{/each}
</div>

<style>
	.toast-controller {
		user-select: none;
		position: absolute;
		display: flex;
		flex-direction: column;

		bottom: 0;
		right: 0;
		padding: 12px 12px 12px 0;
		gap: 8px;
		max-width: 480px;
		z-index: var(--z-blocker);
		overflow-y: auto;
		max-height: 100%;
	}
</style>
