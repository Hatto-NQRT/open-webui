<script lang="ts">
	import { getContext, onMount } from 'svelte';
	import Modal from '../common/Modal.svelte';
	import type { Func } from 'idb/build/util';

	let chat = null;
	const i18n = getContext('i18n');

	export let show = false;
	export let switchToLongModel: Function;
	export let cropLongMessage: Function
	export let confirmToCrop = false;
</script>

<Modal bind:show size="sm">
	<div>
		<div class=" flex justify-between dark:text-gray-300 px-5 pt-4 pb-0.5">
			<div class=" text-lg font-medium self-center">{$i18n.t('Message too long')}</div>
			<button
				class="self-center"
				on:click={() => {
					show = false;
				}}
			>
				<svg
					xmlns="http://www.w3.org/2000/svg"
					viewBox="0 0 20 20"
					fill="currentColor"
					class="w-5 h-5"
				>
					<path
						d="M6.28 5.22a.75.75 0 00-1.06 1.06L8.94 10l-3.72 3.72a.75.75 0 101.06 1.06L10 11.06l3.72 3.72a.75.75 0 101.06-1.06L11.06 10l3.72-3.72a.75.75 0 00-1.06-1.06L10 8.94 6.28 5.22z"
					/>
				</svg>
			</button>
		</div>

		<div class="px-5 pt-4 pb-5 w-full flex flex-col justify-center">
			<div class=" text-sm dark:text-gray-300 mb-1">
				{#if confirmToCrop}
					{$i18n.t(
						"crop_long_message_confirm"
					)}
				{:else}
					{$i18n.t(
						"switch_to_long_model_confirm"
					)}
				{/if}
			</div>

			<div class="flex justify-end">
				<div class="flex flex-col items-end space-x-1 mt-1.5">
					<div class="flex gap-1">
						<button
							class=" self-center px-3.5 py-2 rounded-xl text-sm font-medium bg-gray-100 hover:bg-gray-200 text-gray-800 dark:bg-gray-850 dark:hover:bg-gray-800 dark:text-white"
							type="button"
							on:click={() => {
									show = false;
								}}
						>
							{$i18n.t('Cancel')}
						</button>

						<button
							class=" self-center flex items-center gap-1 px-3.5 py-2 rounded-xl text-sm font-medium bg-emerald-600 hover:bg-emerald-500 text-white"
							type="button"
							id="copy-and-share-chat-button"
							on:click={async () => {
									if (confirmToCrop) {
										// TODO
									} else {
										switchToLongModel();
									}
									show = false;
								}}
						>
							{$i18n.t('Accept')}
						</button>
					</div>
				</div>
			</div>
		</div>
	</div>
</Modal>
