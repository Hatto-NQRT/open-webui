<script lang="ts">
  import { toast } from 'svelte-sonner';
  import fileSaver from 'file-saver';
  const { saveAs } = fileSaver;

  import { onMount, getContext } from 'svelte';
  import {WEBUI_NAME, documents, selectedIndexId, chatType, selectedChatEmbeddingIndex} from '$lib/stores';
  import { createNewDoc, deleteDocByName, getDocs } from '$lib/apis/documents';

  import { SUPPORTED_FILE_TYPE, SUPPORTED_FILE_EXTENSIONS } from '$lib/constants';
  import { uploadDocToVectorDB } from '$lib/apis/rag';
  import { transformFileName } from '$lib/utils';

  import Checkbox from '$lib/components/common/Checkbox.svelte';

  import EditDocModal from '$lib/components/documents/EditDocModal.svelte';
  import AddFilesPlaceholder from '$lib/components/AddFilesPlaceholder.svelte';
  import {deleteEmbeddingFile, embeddingFile, getEmbeddedFiles} from "$lib/apis/embedding";
  import Tooltip from "$lib/components/common/Tooltip.svelte";

  const i18n = getContext('i18n');

  let embeddedFiles = []
  let filesInputElement;
  let inputFiles = '';
  let query = '';

  let params = new URL(document.location.toString()).searchParams;
  selectedIndexId.set(parseInt(params.get('id') || '0'))


  let tags = [];

  let showEditDocModal = false;
  let selectedDoc;
  let selectedTag = '';

  let dragged = false;

  const deleteDoc = async (doc) => {
    await deleteEmbeddingFile($selectedIndexId, doc.id, doc.doc_ref_id);
    embeddedFiles = embeddedFiles.filter(({id}) => id !== doc.id)
  };

  const deleteDocs = async (docs) => {
    const res = await Promise.all(
      docs.map(async (doc) => {
        return await deleteDocByName(localStorage.token, doc.name);
      })
    );

    await documents.set(await getDocs(localStorage.token));
  };

  const uploadDoc = async (file) => {
    const res = await uploadDocToVectorDB(localStorage.token, '', file).catch((error) => {
      toast.error(error);
      return null;
    });

    if (res) {
      await createNewDoc(
        localStorage.token,
        res.collection_name,
        res.filename,
        transformFileName(res.filename),
        res.filename
      ).catch((error) => {
        toast.error(error);
        return null;
      });
      await documents.set(await getDocs(localStorage.token));
    }
  };

  onMount(async () => {
    embeddedFiles = await getEmbeddedFiles($selectedIndexId)
    const dropZone = document.querySelector('body');

    const onDragOver = (e) => {
      e.preventDefault();
      dragged = true;
    };

    const onDragLeave = () => {
      dragged = false;
    };

    const onDrop = async (e) => {
      e.preventDefault();

      if (e.dataTransfer?.files) {
        let reader = new FileReader();

        reader.onload = (event) => {
          files = [
            ...files,
            {
              type: 'image',
              url: `${event.target.result}`
            }
          ];
        };

        const inputFiles = e.dataTransfer?.files;

        if (inputFiles && inputFiles.length > 0) {
          for (const file of inputFiles) {
            console.log(file, file.name.split('.').at(-1));
            if (
              SUPPORTED_FILE_TYPE.includes(file['type']) ||
              SUPPORTED_FILE_EXTENSIONS.includes(file.name.split('.').at(-1))
            ) {
              uploadDoc(file);
            } else {
              toast.error(
                `Unknown File Type '${file['type']}', but accepting and treating as plain text`
              );
              uploadDoc(file);
            }
          }
        } else {
          toast.error($i18n.t(`File not found.`));
        }
      }

      dragged = false;
    };

    dropZone?.addEventListener('dragover', onDragOver);
    dropZone?.addEventListener('drop', onDrop);
    dropZone?.addEventListener('dragleave', onDragLeave);

    return () => {
      dropZone?.removeEventListener('dragover', onDragOver);
      dropZone?.removeEventListener('drop', onDrop);
      dropZone?.removeEventListener('dragleave', onDragLeave);
    };
  });

  let filteredDocs;

  $: filteredDocs = embeddedFiles.filter(
    (doc) => query === '' || doc.file_name.includes(query)
  );
</script>

<svelte:head>
    <title>
        {$i18n.t('Documents')} | {$WEBUI_NAME}
    </title>
</svelte:head>

{#if dragged}
    <div
            class="fixed w-full h-full flex z-50 touch-none pointer-events-none"
            id="dropzone"
            role="region"
            aria-label="Drag and Drop Container"
    >
        <div class="absolute rounded-xl w-full h-full backdrop-blur bg-gray-800/40 flex justify-center">
            <div class="m-auto pt-64 flex flex-col justify-center">
                <div class="max-w-md">
                    <AddFilesPlaceholder>
                        <div class=" mt-2 text-center text-sm dark:text-gray-200 w-full">
                            Drop any files here to add to my documents
                        </div>
                    </AddFilesPlaceholder>
                </div>
            </div>
        </div>
    </div>
{/if}

{#key selectedDoc}
    <EditDocModal bind:show={showEditDocModal} {selectedDoc} />
{/key}

<div class="mb-4">
    <button
            class="flex space-x-1"
            on:click={() => {
			history.back();
		}}
    >
        <div class=" self-center">
            <svg
                    xmlns="http://www.w3.org/2000/svg"
                    viewBox="0 0 20 20"
                    fill="currentColor"
                    class="w-4 h-4"
            >
                <path
                        fill-rule="evenodd"
                        d="M17 10a.75.75 0 01-.75.75H5.612l4.158 3.96a.75.75 0 11-1.04 1.08l-5.5-5.25a.75.75 0 010-1.08l5.5-5.25a.75.75 0 111.04 1.08L5.612 9.25H16.25A.75.75 0 0117 10z"
                        clip-rule="evenodd"
                />
            </svg>
        </div>
        <div class=" self-center font-medium text-sm">{$i18n.t('Back')}</div>
    </button>
</div>

<div class="mb-3">
    <div class="flex justify-between items-center">
        <div class=" text-lg font-semibold self-center">{$i18n.t('Embedded Documents')}</div>
    </div>
</div>

<div class=" flex w-full space-x-2">
    <div class="flex flex-1">
        <div class=" self-center ml-1 mr-3">
            <svg
                    xmlns="http://www.w3.org/2000/svg"
                    viewBox="0 0 20 20"
                    fill="currentColor"
                    class="w-4 h-4"
            >
                <path
                        fill-rule="evenodd"
                        d="M9 3.5a5.5 5.5 0 100 11 5.5 5.5 0 000-11zM2 9a7 7 0 1112.452 4.391l3.328 3.329a.75.75 0 11-1.06 1.06l-3.329-3.328A7 7 0 012 9z"
                        clip-rule="evenodd"
                />
            </svg>
        </div>
        <input
                class=" w-full text-sm pr-4 py-1 rounded-r-xl outline-none bg-transparent"
                bind:value={query}
                placeholder={$i18n.t('Search Documents')}
        />
    </div>

    <div>
        <input
                bind:this={filesInputElement}
                bind:files={inputFiles}
                type="file"
                hidden
                on:change={async () => {
                  await embeddingFile($selectedIndexId, inputFiles[0])
                  embeddedFiles = await getEmbeddedFiles($selectedIndexId)
				}}
        />
        <button
                class=" px-2 py-2 rounded-xl border border-gray-200 dark:border-gray-600 dark:border-0 hover:bg-gray-100 dark:bg-gray-800 dark:hover:bg-gray-700 transition font-medium text-sm flex items-center space-x-1"
                on:click={() => {
				    filesInputElement.click();
			    }}
        >
            <svg
                    xmlns="http://www.w3.org/2000/svg"
                    viewBox="0 0 16 16"
                    fill="currentColor"
                    class="w-4 h-4"
            >
                <path
                        d="M8.75 3.75a.75.75 0 0 0-1.5 0v3.5h-3.5a.75.75 0 0 0 0 1.5h3.5v3.5a.75.75 0 0 0 1.5 0v-3.5h3.5a.75.75 0 0 0 0-1.5h-3.5v-3.5Z"
                />
            </svg>
        </button>
    </div>
</div>

<!-- <div>
    <div
        class="my-3 py-16 rounded-lg border-2 border-dashed dark:border-gray-600 {dragged &&
            ' dark:bg-gray-700'} "
        role="region"
        on:drop={onDrop}
        on:dragover={onDragOver}
        on:dragleave={onDragLeave}
    >
        <div class="  pointer-events-none">
            <div class="text-center dark:text-white text-2xl font-semibold z-50">{$i18n.t('Add Files')}</div>

            <div class=" mt-2 text-center text-sm dark:text-gray-200 w-full">
                Drop any files here to add to my documents
            </div>
        </div>
    </div>
</div> -->

<hr class=" dark:border-gray-850 my-2.5" />

{#if tags.length > 0}
    <div class="px-2.5 pt-1 flex gap-1 flex-wrap">
        <div class="ml-0.5 pr-3 my-auto flex items-center">
            <Checkbox
                    state={filteredDocs.filter((doc) => doc?.selected === 'checked').length ===
				filteredDocs.length
					? 'checked'
					: 'unchecked'}
                    indeterminate={filteredDocs.filter((doc) => doc?.selected === 'checked').length > 0 &&
					filteredDocs.filter((doc) => doc?.selected === 'checked').length !== filteredDocs.length}
                    on:change={(e) => {
					if (e.detail === 'checked') {
						filteredDocs = filteredDocs.map((doc) => ({ ...doc, selected: 'checked' }));
					} else if (e.detail === 'unchecked') {
						filteredDocs = filteredDocs.map((doc) => ({ ...doc, selected: 'unchecked' }));
					}
				}}
            />
        </div>

        {#if filteredDocs.filter((doc) => doc?.selected === 'checked').length === 0}
            <button
                    class="px-2 py-0.5 space-x-1 flex h-fit items-center rounded-full transition bg-gray-50 hover:bg-gray-100 dark:bg-gray-800 dark:text-white"
                    on:click={async () => {
					selectedTag = '';
					// await chats.set(await getChatListByTagName(localStorage.token, tag.name));
				}}
            >
                <div class=" text-xs font-medium self-center line-clamp-1">{$i18n.t('all')}</div>
            </button>

            {#each tags as tag}
                <button
                        class="px-2 py-0.5 space-x-1 flex h-fit items-center rounded-full transition bg-gray-50 hover:bg-gray-100 dark:bg-gray-800 dark:text-white"
                        on:click={async () => {
						selectedTag = tag;
						// await chats.set(await getChatListByTagName(localStorage.token, tag.name));
					}}
                >
                    <div class=" text-xs font-medium self-center line-clamp-1">
                        #{tag}
                    </div>
                </button>
            {/each}
        {:else}
            <div class="flex-1 flex w-full justify-between items-center">
                <div class="text-xs font-medium py-0.5 self-center mr-1">
                    {filteredDocs.filter((doc) => doc?.selected === 'checked').length} Selected
                </div>

                <div class="flex gap-1">
                    <!-- <button
                        class="px-2 py-0.5 space-x-1 flex h-fit items-center rounded-full transition bg-gray-50 hover:bg-gray-100 dark:bg-gray-800 dark:text-white"
                        on:click={async () => {
                            selectedTag = '';
                            // await chats.set(await getChatListByTagName(localStorage.token, tag.name));
                        }}
                    >
                        <div class=" text-xs font-medium self-center line-clamp-1">add tags</div>
                    </button> -->

                    <button
                            class="px-2 py-0.5 space-x-1 flex h-fit items-center rounded-full transition bg-gray-50 hover:bg-gray-100 dark:bg-gray-800 dark:text-white"
                            on:click={async () => {
							deleteDocs(filteredDocs.filter((doc) => doc.selected === 'checked'));
							// await chats.set(await getChatListByTagName(localStorage.token, tag.name));
						}}
                    >
                        <div class=" text-xs font-medium self-center line-clamp-1">
                            {$i18n.t('delete')}
                        </div>
                    </button>
                </div>
            </div>
        {/if}
    </div>
{/if}

<div class="my-3 mb-5">
    {#each filteredDocs as doc}
        <button
                class=" flex space-x-4 cursor-pointer text-left w-full px-3 py-2 dark:hover:bg-white/5 hover:bg-black/5 rounded-xl"
                on:click={() => {
				if (doc?.selected === 'checked') {
					doc.selected = 'unchecked';
				} else {
					doc.selected = 'checked';
				}
			}}
        >
            <div class="my-auto flex items-center">
                <Checkbox state={doc?.selected ?? 'unchecked'} />
            </div>
            <div class=" flex flex-1 space-x-4 cursor-pointer w-full">
                <div class=" flex items-center space-x-3">
                    <div class="p-2.5 bg-red-400 text-white rounded-lg">
                        {#if doc}
                            <svg
                                    xmlns="http://www.w3.org/2000/svg"
                                    viewBox="0 0 24 24"
                                    fill="currentColor"
                                    class="w-6 h-6"
                            >
                                <path
                                        fill-rule="evenodd"
                                        d="M5.625 1.5c-1.036 0-1.875.84-1.875 1.875v17.25c0 1.035.84 1.875 1.875 1.875h12.75c1.035 0 1.875-.84 1.875-1.875V12.75A3.75 3.75 0 0 0 16.5 9h-1.875a1.875 1.875 0 0 1-1.875-1.875V5.25A3.75 3.75 0 0 0 9 1.5H5.625ZM7.5 15a.75.75 0 0 1 .75-.75h7.5a.75.75 0 0 1 0 1.5h-7.5A.75.75 0 0 1 7.5 15Zm.75 2.25a.75.75 0 0 0 0 1.5H12a.75.75 0 0 0 0-1.5H8.25Z"
                                        clip-rule="evenodd"
                                />
                                <path
                                        d="M12.971 1.816A5.23 5.23 0 0 1 14.25 5.25v1.875c0 .207.168.375.375.375H16.5a5.23 5.23 0 0 1 3.434 1.279 9.768 9.768 0 0 0-6.963-6.963Z"
                                />
                            </svg>
                        {:else}
                            <svg
                                    class=" w-6 h-6 translate-y-[0.5px]"
                                    fill="currentColor"
                                    viewBox="0 0 24 24"
                                    xmlns="http://www.w3.org/2000/svg"
                            ><style>
                                .spinner_qM83 {
                                    animation: spinner_8HQG 1.05s infinite;
                                }
                                .spinner_oXPr {
                                    animation-delay: 0.1s;
                                }
                                .spinner_ZTLf {
                                    animation-delay: 0.2s;
                                }
                                @keyframes spinner_8HQG {
                                    0%,
                                    57.14% {
                                        animation-timing-function: cubic-bezier(0.33, 0.66, 0.66, 1);
                                        transform: translate(0);
                                    }
                                    28.57% {
                                        animation-timing-function: cubic-bezier(0.33, 0, 0.66, 0.33);
                                        transform: translateY(-6px);
                                    }
                                    100% {
                                        transform: translate(0);
                                    }
                                }
                            </style><circle class="spinner_qM83" cx="4" cy="12" r="2.5" /><circle
                                    class="spinner_qM83 spinner_oXPr"
                                    cx="12"
                                    cy="12"
                                    r="2.5"
                            /><circle class="spinner_qM83 spinner_ZTLf" cx="20" cy="12" r="2.5" /></svg
                            >
                        {/if}
                    </div>
                    <div class=" self-center flex-1">
                        <div class="flex items-center font-bold line-clamp-1">
                            <span class="mr-4">{doc.file_name}</span>
                            {#if doc.status === 'indexed'}
                                <Tooltip content={$i18n.t('Success')}>
                                    <svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="16" height="16" class="inline-block">
                                        <circle r="8" cx="8" cy="8" style="fill:Green;stroke:gray;stroke-width:0.1" />
                                    </svg>
                                </Tooltip>
                            {/if}
                            {#if doc.status === 'fail'}
                                <Tooltip content={$i18n.t('Fail')}>
                                    <svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="16" height="16" class="inline-block">
                                        <circle r="8" cx="8" cy="8" style="fill:Red;stroke:gray;stroke-width:0.1" />
                                    </svg>
                                </Tooltip>
                            {/if}
                            {#if doc.status === 'pending'}
                                <Tooltip content={$i18n.t('Pending')}>
                                    <svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="16" height="16" class="inline-block">
                                        <circle r="8" cx="8" cy="8" style="fill:Gray;stroke:gray;stroke-width:0.1" />
                                    </svg>
                                </Tooltip>
                            {/if}
                        </div>
                        <div class=" text-xs overflow-hidden text-ellipsis line-clamp-1">
                            {new Date(doc.created * 1000)}
                        </div>
                    </div>
                </div>
            </div>
            <div class="flex flex-row space-x-1 self-center">
                <!-- <button
            class="self-center w-fit text-sm px-2 py-2 border dark:border-gray-600 rounded-xl"
            type="button"
            on:click={() => {
                console.log('download file');
            }}
        >
            <svg
                xmlns="http://www.w3.org/2000/svg"
                viewBox="0 0 16 16"
                fill="currentColor"
                class="w-4 h-4"
            >
                <path
                    d="M8.75 2.75a.75.75 0 0 0-1.5 0v5.69L5.03 6.22a.75.75 0 0 0-1.06 1.06l3.5 3.5a.75.75 0 0 0 1.06 0l3.5-3.5a.75.75 0 0 0-1.06-1.06L8.75 8.44V2.75Z"
                />
                <path
                    d="M3.5 9.75a.75.75 0 0 0-1.5 0v1.5A2.75 2.75 0 0 0 4.75 14h6.5A2.75 2.75 0 0 0 14 11.25v-1.5a.75.75 0 0 0-1.5 0v1.5c0 .69-.56 1.25-1.25 1.25h-6.5c-.69 0-1.25-.56-1.25-1.25v-1.5Z"
                />
            </svg>
        </button> -->

                <button
                        class="self-center w-fit text-sm px-2 py-2 dark:text-gray-300 dark:hover:text-white hover:bg-black/5 dark:hover:bg-white/5 rounded-xl"
                        type="button"
                        on:click={(e) => {
						e.stopPropagation();
						deleteDoc(doc);
					}}
                >
                    <svg
                            xmlns="http://www.w3.org/2000/svg"
                            fill="none"
                            viewBox="0 0 24 24"
                            stroke-width="1.5"
                            stroke="currentColor"
                            class="w-4 h-4"
                    >
                        <path
                                stroke-linecap="round"
                                stroke-linejoin="round"
                                d="M14.74 9l-.346 9m-4.788 0L9.26 9m9.968-3.21c.342.052.682.107 1.022.166m-1.022-.165L18.16 19.673a2.25 2.25 0 01-2.244 2.077H8.084a2.25 2.25 0 01-2.244-2.077L4.772 5.79m14.456 0a48.108 48.108 0 00-3.478-.397m-12 .562c.34-.059.68-.114 1.022-.165m0 0a48.11 48.11 0 013.478-.397m7.5 0v-.916c0-1.18-.91-2.164-2.09-2.201a51.964 51.964 0 00-3.32 0c-1.18.037-2.09 1.022-2.09 2.201v.916m7.5 0a48.667 48.667 0 00-7.5 0"
                        />
                    </svg>
                </button>
            </div>
        </button>
    {/each}
</div>
