<script lang="ts">
	import { v4 as uuidv4 } from 'uuid';
	import { toast } from 'svelte-sonner';

	import { onMount, tick, getContext } from 'svelte';
	import { goto } from '$app/navigation';
	import { page } from '$app/stores';

	import {
		models,
		modelfiles,
		user,
		settings,
		chats,
		chatId,
		config,
		WEBUI_NAME,
		tags as _tags,
		showSidebar, chatType, promptOptions, selectedChatEmbeddingIndex, isNewChat
	} from '$lib/stores';
	import { copyToClipboard, splitStream } from '$lib/utils';

	import { generateChatCompletion, cancelOllamaRequest } from '$lib/apis/ollama';
	import {
		addTagById,
		createNewChat,
		deleteTagById,
		getAllChatTags,
		getChatList,
		getTagsById,
		updateChatById
	} from '$lib/apis/chats';
	import { queryCollection, queryDoc } from '$lib/apis/rag';
	import { generateOpenAIChatCompletion, generateTitle } from '$lib/apis/openai';

	import MessageInput from '$lib/components/chat/MessageInput.svelte';
	import Messages from '$lib/components/chat/Messages.svelte';
	import ModelSelector from '$lib/components/chat/ModelSelector.svelte';
	import Navbar from '$lib/components/layout/Navbar.svelte';
	import { RAGTemplate } from '$lib/utils/rag';
	import { LITELLM_API_BASE_URL, OLLAMA_API_BASE_URL, OPENAI_API_BASE_URL } from '$lib/constants';
	import { WEBUI_BASE_URL } from '$lib/constants';
	import { createOpenAITextStream } from '$lib/apis/streaming';
	import { queryMemory } from '$lib/apis/memories';
	import {
		createChatCompletionApiMessages, getAbstractPrompt, getDefaultParams,
		getSystemPrompt,
		getUserPrompt
	} from "$lib/utils/prompt_utils";
	import ChatSessionSettingModal from "$lib/components/chat/ChatSessionSettingModal.svelte";
	import {highlightText} from "$lib/apis/documents";
	import {queryRankedChunk} from "$lib/apis/embedding";
	import { chunkText } from '$lib/apis/tools';

	const i18n = getContext('i18n');

	let stopResponseFlag = false;
	let autoScroll = true;
	let processing = '';
	let messagesContainerElement: HTMLDivElement;
	let currentRequestId = null;

	let showModelSelector = true;
	let showSessionSetting = false;

	let selectedModels = [''];
	let atSelectedModel = '';

	let sessionConfig = {}

	let selectedModelfile = null;
	$: selectedModelfile =
		selectedModels.length === 1 &&
		$modelfiles.filter((modelfile) => modelfile.tagName === selectedModels[0]).length > 0
			? $modelfiles.filter((modelfile) => modelfile.tagName === selectedModels[0])[0]
			: null;

	let selectedModelfiles = {};
	$: selectedModelfiles = selectedModels.reduce((a, tagName, i, arr) => {
		const modelfile =
			$modelfiles.filter((modelfile) => modelfile.tagName === tagName)?.at(0) ?? undefined;

		return {
			...a,
			...(modelfile && { [tagName]: modelfile })
		};
	}, {});

	$: (async () => {
		if ($selectedChatEmbeddingIndex && $chatId) {
			await updateChatById(localStorage.token, $chatId, {embedding_index_id: $selectedChatEmbeddingIndex})
		}
	})()

	let params = new URL(document.location.toString()).searchParams;
	chatType.set(params.get('type') || 'chat')
	let currentChatType = $chatType
	let chat = null;
	let tags = [];

	let title = '';
	let prompt = '';
	let files = [];
	let messages = [];
	let history = {
		messages: {},
		currentId: null
	};


	$: if (!!history.currentId && !$isNewChat) {
		let _messages = [];

		let currentMessage = history.messages[history.currentId];
		if (!!currentMessage) {
			while (!!currentMessage) {
				_messages.unshift({ ...currentMessage });
				currentMessage =
					currentMessage.parentId !== null ? history.messages[currentMessage.parentId] : null;
			}
			messages = _messages;
		}
	} else {
		isNewChat.set(false)
		history = {
			messages: {},
			currentId: null
		}
		messages = [];
	}

	$: (() => {
		if (currentChatType === $chatType && !!selectedModels?.[0]) {
			return
		}
		currentChatType = $chatType
		if ($page.url.searchParams.get('models')) {
			selectedModels = $page.url.searchParams.get('models')?.split(',');
		} else if ($settings?.models) {
			selectedModels = $settings?.models;
		} else if ($config?.default_models) {
			selectedModels = $config?.default_models.split(',');
		} else {
			selectedModels = [$models[$config.chat_type_model_map?.[$chatType] || 0]?.id];
		}
		sessionConfig = {
			system: getSystemPrompt($chatType),
			model_params: {...$models.find((model) => model.id === selectedModels[0])?.default_params, ...getDefaultParams($chatType)}
		}
	})()

	onMount(async () => {
		await initNewChat();
	});

	//////////////////////////
	// Web functions
	//////////////////////////

	const initNewChat = async () => {
		if (currentRequestId !== null) {
			await cancelOllamaRequest(localStorage.token, currentRequestId);
			currentRequestId = null;
		}
		window.history.replaceState(history.state, '', `/?type=${$chatType}`);
		await chatId.set('');

		autoScroll = true;

		title = '';
		messages = [];
		history = {
			messages: {},
			currentId: null
		};

		if ($page.url.searchParams.get('q')) {
			prompt = $page.url.searchParams.get('q') ?? '';
			if (prompt) {
				await tick();
				submitPrompt(prompt);
			}
		}

		selectedModels = selectedModels.map((modelId) =>
			$models.map((m) => m.id).includes(modelId) ? modelId : ''
		);

		let _settings = JSON.parse(localStorage.getItem('settings') ?? '{}');
		settings.set({
			..._settings
		});

		const chatInput = document.getElementById('chat-textarea');
		setTimeout(() => chatInput?.focus(), 0);
	};

	const scrollToBottom = async () => {
		await tick();
		if (messagesContainerElement) {
			messagesContainerElement.scrollTop = messagesContainerElement.scrollHeight;
		}
	};

	const addNotFoundResponseMessage = async (userMessageId: string) => {
		let responseMessageId = uuidv4();
		let responseMessage = {
			parentId: userMessageId,
			id: responseMessageId,
			childrenIds: [],
			role: 'assistant',
			content: 'Không có dữ liệu nào liên quan câu hỏi này.',
			model: selectedModels[0],
			userContext: null,
			done: true,
			timestamp: Math.floor(Date.now() / 1000) // Unix epoch
		};

		// Add message to history and Set currentId to messageId
		history.messages[responseMessageId] = responseMessage;
		history.currentId = responseMessageId;

		// Append messageId to childrenIds of parent message
		if (userMessageId !== null) {
			history.messages[userMessageId].childrenIds = [
				...history.messages[userMessageId].childrenIds,
				responseMessageId
			];
		}
		if ($settings.saveChatHistory ?? true) {
			chat = await updateChatById(localStorage.token, $chatId, {
				messages: messages,
				history: history
			});
		}
	}

	//////////////////////////
	// Ollama functions
	//////////////////////////

	const submitPrompt = async (userPrompt, _user = null) => {
		console.log('submitPrompt', $chatId);

		selectedModels = selectedModels.map((modelId) =>
			$models.map((m) => m.id).includes(modelId) ? modelId : ''
		);

		if (selectedModels.includes('')) {
			toast.error($i18n.t('Model not selected'));
		} else if (messages.length != 0 && messages.at(-1).done != true) {
			// Response not done
			console.log('wait');
		} else if (
			files.length > 0 &&
			files.filter((file) => file.upload_status === false).length > 0
		) {
			// Upload not done
			toast.error(
				$i18n.t(
					`Oops! Hold tight! Your files are still in the processing oven. We're cooking them up to perfection. Please be patient and we'll let you know once they're ready.`
				)
			);
		} else {
			// Reset chat message textarea height
			document.getElementById('chat-textarea').style.height = '';

			// Create user message
			let userMessageId = uuidv4();
			let userMessage = {
				id: userMessageId,
				parentId: messages.length !== 0 ? messages.at(-1).id : null,
				childrenIds: [],
				role: 'user',
				user: _user ?? undefined,
				// content: getUserPrompt($chatType, userPrompt, $promptOptions),
				content: userPrompt,
				files: files.length > 0 ? files : undefined,
				models: selectedModels.filter((m, mIdx) => selectedModels.indexOf(m) === mIdx),
				timestamp: Math.floor(Date.now() / 1000) // Unix epoch
			};

			// Add message to history and Set currentId to messageId
			history.messages[userMessageId] = userMessage;
			history.currentId = userMessageId;

			// Append messageId to childrenIds of parent message
			if (messages.length !== 0) {
				history.messages[messages.at(-1).id].childrenIds.push(userMessageId);
			}

			// Wait until history/message have been updated
			await tick();

			// Create new chat if only one message in messages
			if (messages.length == 1) {
				if ($settings.saveChatHistory ?? true) {
					chat = await createNewChat(localStorage.token, {
						id: $chatId,
						title: $i18n.t('New Chat'),
						models: selectedModels,
						chat_type: $chatType,
						embedding_index_id: $selectedChatEmbeddingIndex,
						system: sessionConfig.system, // $settings.system ?? undefined,
						options: sessionConfig.model_params,
						messages: messages,
						history: history,
						tags: [],
						timestamp: Date.now()
					});
					await chats.set(await getChatList(localStorage.token));
					await chatId.set(chat.id);
				} else {
					await chatId.set('local');
				}
				await tick();
			}

			// Reset chat input textarea
			prompt = '';
			files = [];

			if ($chatType === 'chat_embedding') {
				const chunks = await queryRankedChunk($selectedChatEmbeddingIndex, userPrompt)
				if (!chunks || chunks.length === 0) {
					await addNotFoundResponseMessage(userMessageId)
				} else {
					chunks.sort((chunk1, chunk2) => chunk1.score > chunk2.score ? 1 : -1)
					await Promise.all(chunks.map((chunk) => {
						const citations = [{
							source: {
								type: 'doc',
								name: chunk.metadata.file_name,
							},
							document: [chunk.text],
							score: chunk.score
						}]
						const prompt = getAbstractPrompt(chunk.text, userPrompt)
						return sendPrompt(prompt, userMessageId, null, citations);
					}))

					if (!history.messages[userMessageId]?.childrenIds?.length) {
						await addNotFoundResponseMessage(userMessageId)
					}
				}
			} else {
				// Send prompt
				await sendPrompt(userPrompt, userMessageId);
			}

			if (messages.length == 2) {
				window.history.replaceState(history.state, '', `/c/${$chatId}`);

				const _title = await generateChatTitle(userPrompt);
				await setChatTitle($chatId, _title);
			}
		}
	};

	const sendPrompt = async (prompt, parentId, modelId = null, citations = null) => {
		const _chatId = JSON.parse(JSON.stringify($chatId));

		await Promise.all(
			(modelId ? [modelId] : atSelectedModel !== '' ? [atSelectedModel.id] : selectedModels).map(
				async (modelId) => {
					console.log('modelId', modelId);
					const model = $models.filter((m) => m.id === modelId).at(0);

					if (model) {
						// Create response message
						let responseMessageId = uuidv4();
						let responseMessage = {
							parentId: parentId,
							id: responseMessageId,
							childrenIds: [],
							role: 'assistant',
							content: '',
							model: model.id,
							userContext: null,
							timestamp: Math.floor(Date.now() / 1000) // Unix epoch
						};

						if (citations) {
							responseMessage.citations = citations
						}

						// Add message to history and Set currentId to messageId
						history.messages[responseMessageId] = responseMessage;
						history.currentId = responseMessageId;

						// Append messageId to childrenIds of parent message
						if (parentId !== null) {
							history.messages[parentId].childrenIds = [
								...history.messages[parentId].childrenIds,
								responseMessageId
							];
						}

						await tick();

						let userContext = null;
						if ($settings?.memory ?? false) {
							if (userContext === null) {
								const res = await queryMemory(localStorage.token, prompt).catch((error) => {
									toast.error(error);
									return null;
								});

								if (res) {
									if (res.documents[0].length > 0) {
										userContext = res.documents.reduce((acc, doc, index) => {
											const createdAtTimestamp = res.metadatas[index][0].created_at;
											const createdAtDate = new Date(createdAtTimestamp * 1000)
												.toISOString()
												.split('T')[0];
											acc.push(`${index + 1}. [${createdAtDate}]. ${doc[0]}`);
											return acc;
										}, []);
									}

									console.log(userContext);
								}
							}
						}
						responseMessage.userContext = userContext;

						if (model?.external) {
							await sendPromptOpenAI(model, prompt, responseMessageId, _chatId);
						} else if (model) {
							await sendPromptOllama(model, prompt, responseMessageId, _chatId);
						}
					} else {
						toast.error($i18n.t(`Model {{modelId}} not found`, { modelId }));
					}
				}
			)
		);

		await chats.set(await getChatList(localStorage.token));
	};

	const sendPromptOllama = async (model, userPrompt, responseMessageId, _chatId) => {
		model = model.id;
		const responseMessage = history.messages[responseMessageId];

		// Wait until history/message have been updated
		await tick();

		// Scroll down
		scrollToBottom();

		const messagesBody = [
			$settings.system || (responseMessage?.userContext ?? null)
				? {
						role: 'system',
						content:
							$settings.system + (responseMessage?.userContext ?? null)
								? `\n\nUser Context:\n${responseMessage.userContext.join('\n')}`
								: ''
				  }
				: undefined,
			...messages
		]
			.filter((message) => message)
			.map((message, idx, arr) => {
				// Prepare the base message object
				const baseMessage = {
					role: message.role,
					content: arr.length - 2 !== idx ? message.content : message?.raContent ?? message.content
				};

				// Extract and format image URLs if any exist
				const imageUrls = message.files
					?.filter((file) => file.type === 'image')
					.map((file) => file.url.slice(file.url.indexOf(',') + 1));

				// Add images array only if it contains elements
				if (imageUrls && imageUrls.length > 0 && message.role === 'user') {
					baseMessage.images = imageUrls;
				}

				return baseMessage;
			});

		let lastImageIndex = -1;

		// Find the index of the last object with images
		messagesBody.forEach((item, index) => {
			if (item.images) {
				lastImageIndex = index;
			}
		});

		// Remove images from all but the last one
		messagesBody.forEach((item, index) => {
			if (index !== lastImageIndex) {
				delete item.images;
			}
		});

		const docs = messages
			.filter((message) => message?.files ?? null)
			.map((message) =>
				message.files.filter((item) => item.type === 'doc' || item.type === 'collection')
			)
			.flat(1);

		const [res, controller] = await generateChatCompletion(localStorage.token, {
			model: model,
			messages: messagesBody,
			options: {
				...($settings.options ?? {}),
				stop:
					$settings?.options?.stop ?? undefined
						? $settings.options.stop.map((str) =>
								decodeURIComponent(JSON.parse('"' + str.replace(/\"/g, '\\"') + '"'))
						  )
						: undefined
			},
			format: $settings.requestFormat ?? undefined,
			keep_alive: $settings.keepAlive ?? undefined,
			docs: docs.length > 0 ? docs : undefined,
			citations: docs.length > 0
		});

		if (res && res.ok) {
			console.log('controller', controller);

			const reader = res.body
				.pipeThrough(new TextDecoderStream())
				.pipeThrough(splitStream('\n'))
				.getReader();

			while (true) {
				const { value, done } = await reader.read();
				if (done || stopResponseFlag || _chatId !== $chatId) {
					responseMessage.done = true;
					messages = messages;

					if (stopResponseFlag) {
						controller.abort('User: Stop Response');
						await cancelOllamaRequest(localStorage.token, currentRequestId);
					}

					currentRequestId = null;

					break;
				}

				try {
					let lines = value.split('\n');

					for (const line of lines) {
						if (line !== '') {
							console.log(line);
							let data = JSON.parse(line);

							if ('citations' in data) {
								responseMessage.citations = data.citations;
								continue;
							}

							if ('detail' in data) {
								throw data;
							}

							if ('id' in data) {
								console.log(data);
								currentRequestId = data.id;
							} else {
								if (data.done == false) {
									if (responseMessage.content == '' && data.message.content == '\n') {
										continue;
									} else {
										responseMessage.content += data.message.content;
										messages = messages;
									}
								} else {
									responseMessage.done = true;

									if (responseMessage.content == '') {
										responseMessage.error = true;
										responseMessage.content =
											'Oops! No text generated from Ollama, Please try again.';
									}

									responseMessage.context = data.context ?? null;
									responseMessage.info = {
										total_duration: data.total_duration,
										load_duration: data.load_duration,
										sample_count: data.sample_count,
										sample_duration: data.sample_duration,
										prompt_eval_count: data.prompt_eval_count,
										prompt_eval_duration: data.prompt_eval_duration,
										eval_count: data.eval_count,
										eval_duration: data.eval_duration
									};
									messages = messages;

									if ($settings.notificationEnabled && !document.hasFocus()) {
										const notification = new Notification(
											selectedModelfile
												? `${
														selectedModelfile.title.charAt(0).toUpperCase() +
														selectedModelfile.title.slice(1)
												  }`
												: `${model}`,
											{
												body: responseMessage.content,
												icon: selectedModelfile?.imageUrl ?? `${WEBUI_BASE_URL}/static/favicon.png`
											}
										);
									}

									if ($settings.responseAutoCopy) {
										copyToClipboard(responseMessage.content);
									}

									if ($settings.responseAutoPlayback) {
										await tick();
										document.getElementById(`speak-button-${responseMessage.id}`)?.click();
									}
								}
							}
						}
					}
				} catch (error) {
					console.log(error);
					if ('detail' in error) {
						toast.error(error.detail);
					}
					break;
				}

				if (autoScroll) {
					scrollToBottom();
				}
			}

			if ($chatId == _chatId) {
				if ($settings.saveChatHistory ?? true) {
					chat = await updateChatById(localStorage.token, _chatId, {
						messages: messages,
						history: history
					});
					await chats.set(await getChatList(localStorage.token));
				}
			}
		} else {
			if (res !== null) {
				const error = await res.json();
				console.log(error);
				if ('detail' in error) {
					toast.error(error.detail);
					responseMessage.content = error.detail;
				} else {
					toast.error(error.error);
					responseMessage.content = error.error;
				}
			} else {
				toast.error(
					$i18n.t(`Uh-oh! There was an issue connecting to {{provider}}.`, { provider: 'Ollama' })
				);
				responseMessage.content = $i18n.t(`Uh-oh! There was an issue connecting to {{provider}}.`, {
					provider: 'Ollama'
				});
			}

			responseMessage.error = true;
			responseMessage.content = $i18n.t(`Uh-oh! There was an issue connecting to {{provider}}.`, {
				provider: 'Ollama'
			});
			responseMessage.done = true;
			messages = messages;
		}

		stopResponseFlag = false;
		await tick();

		if (autoScroll) {
			scrollToBottom();
		}

		if (messages.length == 2 && messages.at(1).content !== '') {
			window.history.replaceState(history.state, '', `/c/${_chatId}`);
			const _title = await generateChatTitle(userPrompt);
			await setChatTitle(_chatId, _title);
		}
	};

	const sendPromptOpenAI = async (model, userPrompt, responseMessageId, _chatId) => {
		const responseMessage = history.messages[responseMessageId];
		const docs = messages
			.filter((message) => message?.files ?? null)
			.map((message) =>
				message.files.filter((item) => item.type === 'doc' || item.type === 'collection')
			)
			.flat(1);

		console.log(docs);

		scrollToBottom();

		let prompts = [userPrompt]
		if ($chatType.startsWith('translate')) {
			prompts = await chunkText(selectedModels[0], $chatType, userPrompt, 100)
		}
		
		let trailingChar = '';
		for (let subPrompt of prompts) {
			try {
				const {apiMessages, citations} = await createChatCompletionApiMessages($chatType, sessionConfig.system, subPrompt, $selectedChatEmbeddingIndex, messages, $promptOptions)
				if (citations) {
					responseMessage.citations = citations
				}

				const [res, controller] = await generateOpenAIChatCompletion(
					localStorage.token,
					{
						model: model.id,
						stream: true,
						messages: apiMessages,
						seed: sessionConfig?.model_params?.seed ?? undefined,
						stop:
							$settings?.options?.stop ?? undefined
								? $settings.options.stop.map((str) =>
									decodeURIComponent(JSON.parse('"' + str.replace(/\"/g, '\\"') + '"'))
								)
								: undefined,
						temperature: sessionConfig?.model_params?.temperature ?? undefined,
						top_p: sessionConfig?.model_params?.top_p ?? undefined,
						num_ctx: sessionConfig?.model_params?.num_ctx ?? undefined,
						frequency_penalty: sessionConfig?.model_params?.repetition_penalty ?? undefined,
						max_tokens: sessionConfig?.model_params?.num_predict ?? undefined,
						docs: docs.length > 0 ? docs : undefined,
						citations: docs.length > 0
					},
					model?.source?.toLowerCase() === 'litellm'
						? `${LITELLM_API_BASE_URL}/v1`
						: `${OPENAI_API_BASE_URL}`
				);

				// Wait until history/message have been updated
				await tick();

				scrollToBottom();

				if (res && res.ok && res.body) {
					const textStream = await createOpenAITextStream(res.body, $settings.splitLargeChunks);

					let atChunkBeginning = true	
					for await (const update of textStream) {
						const { value, done, citations, error } = update;
						if (error) {
							await handleOpenAIError(error, null, model, responseMessage);
							break;
						}
						if (done || stopResponseFlag || _chatId !== $chatId) {
							if (
								$chatType === 'chat_embedding' &&
								responseMessage.content.includes('Đoạn văn này không có nội dung bạn muốn tìm')
							) {
								if (responseMessage.parentId && history.messages[responseMessage.parentId]?.childrenIds.length > 1) {
									delete history.messages[responseMessageId];
									history.messages[responseMessage.parentId].childrenIds = history.messages[responseMessage.parentId].childrenIds.filter(_id => _id !== responseMessageId);
									const l = history.messages[responseMessage.parentId].childrenIds.length;
									history.currentId = history.messages[responseMessage.parentId].childrenIds[l - 1]
								} else {
									responseMessage.content = 'Không có dữ liệu nào liên quan câu hỏi này.'
									responseMessage.citations = null
								}
								messages = messages;
								break
							}

							// responseMessage.done = true;
							// messages = messages;

							if (stopResponseFlag) {
								controller.abort('User: Stop Response');
							}

							if ($chatType === 'chat_embedding') {
								const highlightedContent = await highlightText(responseMessage.citations[0].document[0], responseMessage.content)
								responseMessage.citations[0].document[0] = highlightedContent
							}

							break;
						}

						if (citations) {
							responseMessage.citations = citations;
							continue;
						}

						if (responseMessage.content == '' && value == '\n') {
							continue;
						} else {
							if (atChunkBeginning) {
								responseMessage.content += trailingChar + value;
								atChunkBeginning = false;
								trailingChar = '';
							}
							else {
								responseMessage.content += value;
							}
							// responseMessage.content += value;
							messages = messages;
						}
					}
					trailingChar = subPrompt.charAt(subPrompt.length - 1);
					if (trailingChar === '。') trailingChar = ' ';
				} else {
					await handleOpenAIError(null, res, model, responseMessage);
				}
			} catch (error) {
				await handleOpenAIError(error, null, model, responseMessage);
			}
		}

		responseMessage.done = true;
		messages = messages;

		if ($settings.notificationEnabled && !document.hasFocus()) {
			const notification = new Notification(`OpenAI ${model}`, {
				body: responseMessage.content,
				icon: `${WEBUI_BASE_URL}/static/favicon.png`
			});
		}

		if ($settings.responseAutoCopy) {
			copyToClipboard(responseMessage.content);
		}

		if ($settings.responseAutoPlayback) {
			await tick();
			document.getElementById(`speak-button-${responseMessage.id}`)?.click();
		}

		if (autoScroll) {
			scrollToBottom();
		}

		if ($chatId == _chatId) {
			if ($settings.saveChatHistory ?? true) {
				chat = await updateChatById(localStorage.token, _chatId, {
					messages: messages,
					history: history
				});
				await chats.set(await getChatList(localStorage.token));
			}
		}

		messages = messages;

		stopResponseFlag = false;
		await tick();

		if (autoScroll) {
			scrollToBottom();
		}
	};

	const handleOpenAIError = async (error, res: Response | null, model, responseMessage) => {
		let errorMessage = '';
		let innerError;

		if (error) {
			innerError = error;
		} else if (res !== null) {
			innerError = await res.json();
		}
		console.error(innerError);
		if ('detail' in innerError) {
			toast.error(innerError.detail);
			errorMessage = innerError.detail;
		} else if ('error' in innerError) {
			if ('message' in innerError.error) {
				toast.error(innerError.error.message);
				errorMessage = innerError.error.message;
			} else {
				toast.error(innerError.error);
				errorMessage = innerError.error;
			}
		} else if ('message' in innerError) {
			toast.error(innerError.message);
			errorMessage = innerError.message;
		}

		responseMessage.error = true;
		responseMessage.content =
			$i18n.t(`Uh-oh! There was an issue connecting to {{provider}}.`, {
				provider: model.name ?? model.id
			}) +
			'\n' +
			errorMessage;
		responseMessage.done = true;

		messages = messages;
	};

	const stopResponse = () => {
		stopResponseFlag = true;
		console.log('stopResponse');
	};

	const regenerateResponse = async (message) => {
		console.log('regenerateResponse');

		if (messages.length != 0) {
			let userMessage = history.messages[message.parentId];
			let userPrompt = userMessage.content;

			if ((userMessage?.models ?? [...selectedModels]).length == 1) {
				await sendPrompt(userPrompt, userMessage.id);
			} else {
				await sendPrompt(userPrompt, userMessage.id, message.model);
			}
		}
	};

	const continueGeneration = async () => {
		console.log('continueGeneration');
		const _chatId = JSON.parse(JSON.stringify($chatId));

		if (messages.length != 0 && messages.at(-1).done == true) {
			const responseMessage = history.messages[history.currentId];
			responseMessage.done = false;
			await tick();

			const model = $models.filter((m) => m.id === responseMessage.model).at(0);

			if (model) {
				if (model?.external) {
					await sendPromptOpenAI(
						model,
						history.messages[responseMessage.parentId].content,
						responseMessage.id,
						_chatId
					);
				} else
					await sendPromptOllama(
						model,
						history.messages[responseMessage.parentId].content,
						responseMessage.id,
						_chatId
					);
			}
		} else {
			toast.error($i18n.t(`Model {{modelId}} not found`, { modelId }));
		}
	};

	const generateChatTitle = async (userPrompt) => {
		if ($settings?.title?.auto ?? true) {
			const model = $models.find((model) => model.id === selectedModels[0]);

			const titleModelId =
				model?.external ?? false
					? $settings?.title?.modelExternal ?? selectedModels[0]
					: $settings?.title?.model ?? selectedModels[0];
			const titleModel = $models.find((model) => model.id === titleModelId);

			console.log(titleModel);
			const title = await generateTitle(
				localStorage.token,
				$settings?.title?.prompt ??
					$i18n.t(
						"Create a concise, 3-5 word phrase as a header for the following query, strictly adhering to the 3-5 word limit and avoiding the use of the word 'title':"
					) + ' {{prompt}}',
				titleModelId,
				userPrompt,
				titleModel?.external ?? false
					? titleModel?.source?.toLowerCase() === 'litellm'
						? `${LITELLM_API_BASE_URL}/v1`
						: `${OPENAI_API_BASE_URL}`
					: `${OLLAMA_API_BASE_URL}/v1`
			);

			return title;
		} else {
			return `${userPrompt}`;
		}
	};

	const setChatTitle = async (_chatId, _title) => {
		if (_chatId === $chatId) {
			title = _title;
		}

		if ($settings.saveChatHistory ?? true) {
			chat = await updateChatById(localStorage.token, _chatId, { title: _title });
			await chats.set(await getChatList(localStorage.token));
		}
	};

	const getTags = async () => {
		return await getTagsById(localStorage.token, $chatId).catch(async (error) => {
			return [];
		});
	};

	const addTag = async (tagName) => {
		const res = await addTagById(localStorage.token, $chatId, tagName);
		tags = await getTags();

		chat = await updateChatById(localStorage.token, $chatId, {
			tags: tags
		});

		_tags.set(await getAllChatTags(localStorage.token));
	};

	const deleteTag = async (tagName) => {
		const res = await deleteTagById(localStorage.token, $chatId, tagName);
		tags = await getTags();

		chat = await updateChatById(localStorage.token, $chatId, {
			tags: tags
		});

		_tags.set(await getAllChatTags(localStorage.token));
	};
</script>

<svelte:head>
	<title>
		{title
			? `${title.length > 30 ? `${title.slice(0, 30)}...` : title} | ${$WEBUI_NAME}`
			: `${$WEBUI_NAME}`}
	</title>
</svelte:head>

{#if showSessionSetting}
<ChatSessionSettingModal
		bind:show={showSessionSetting}
		bind:config={sessionConfig}
/>
{/if}

<div
	class="min-h-screen max-h-screen {$showSidebar
		? 'md:max-w-[calc(100%-260px)]'
		: ''} w-full max-w-full flex flex-col"
>
	<Navbar
		{title}
		bind:selectedModels
		bind:showModelSelector
		shareEnabled={messages.length > 0}
		sessionSettingHandler={() => {
			showSessionSetting = true
		}}
		{chat}
		{initNewChat}
	/>
	<div class="flex flex-col flex-auto">
		<div
			class=" pb-2.5 flex flex-col justify-between w-full flex-auto overflow-auto h-0 max-w-full"
			id="messages-container"
			bind:this={messagesContainerElement}
			on:scroll={(e) => {
				autoScroll =
					messagesContainerElement.scrollHeight - messagesContainerElement.scrollTop <=
					messagesContainerElement.clientHeight + 5;
			}}
		>
			<div class=" h-full w-full flex flex-col pt-2 pb-4">
				<Messages
					chatId={$chatId}
					{selectedModels}
					{selectedModelfiles}
					{processing}
					bind:history
					bind:messages
					bind:autoScroll
					bind:prompt
					bottomPadding={files.length > 0}
					suggestionPrompts={selectedModelfile?.suggestionPrompts ??
						$config.default_prompt_suggestions}
					{sendPrompt}
					{continueGeneration}
					{regenerateResponse}
				/>
			</div>
		</div>
	</div>
</div>

<MessageInput
	bind:files
	bind:prompt
	bind:autoScroll
	selectedModel={{name: selectedModels[0]}}
	{messages}
	{submitPrompt}
	{stopResponse}
/>
