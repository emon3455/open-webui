<script>
	import { onMount, afterUpdate } from 'svelte';
	import axios from 'axios';
	import ReactMarkdown from 'react-markdown';
	// import rehypeRaw from 'rehype-raw';
	import MarkdownRenderer from './MarkdownRenderer.svelte';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import Spinner from '../common/Spinner.svelte';
	import { Plus, FileText, X, ChevronDown, LoaderCircle } from 'lucide-svelte';
	import FileItem from './FileItem.svelte';

	// State variables
	let message = '';
	let messages = [];
	let loading = false;
	let isInitialMessageSent = false;
	let publicModels = [];
	let isDropdownOpen = false;
	let modelsLoaded = false;
	let selectedModel = null;
	let groupID = null;
	let selectedFiles = [];
	let showChat = false; // Control visibility of chat interface

	// Refs
	let inputRef;
	let fileInputRef;
	let dropdownRef;
	let chatInputArea;

	const authToken =
		'Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6ImQyMmQ2Mjg5LTcyZjMtNDZjOC05MzgzLWU0OGNhOWU2NWY2YiJ9.jnETT0RMeNYv81bJfbxbm4fpqSobmiBsxGgUJIm9YzI';

	onMount(() => {
		fetchGroupID();
	});

	afterUpdate(() => {
		// Scroll to bottom of chat
		const chatArea = document.getElementById('chatArea');
		if (chatArea) {
			chatArea.scrollTop = chatArea.scrollHeight;
		}
	});

	const fetchGroupID = async () => {
		try {
			const response = await axios.get('https://ai.optimalmd.com/api/v1/groups/', {
				headers: { Authorization: authToken }
			});
			const publicPlan = response.data.find((group) => group.name === 'Public Plan');
			if (publicPlan) {
				groupID = publicPlan.id;
				fetchPublicModels(publicPlan.id);
			}
		} catch (error) {
			console.error('Error fetching group ID:', error);
		}
	};

	const fetchPublicModels = async (groupID) => {
		try {
			const response = await axios.get('https://ai.optimalmd.com/api/models', {
				headers: { Authorization: authToken }
			});
			const filteredModels = response.data.data.filter((model) =>
				model.info.access_control?.read.group_ids.includes(groupID)
			);
			publicModels = filteredModels;
			if (filteredModels.length > 0) {
				selectedModel = filteredModels[0];
			}
			modelsLoaded = true;
		} catch (error) {
			console.error('Error fetching public models:', error);
			modelsLoaded = true;
		}
	};

	const uploadFile = async (file) => {
		const formData = new FormData();
		formData.append('file', file.localFile);

		try {
			selectedFiles = selectedFiles.map((f) => (f.id === file.id ? { ...f, isLoading: true } : f));

			const response = await axios.post('https://ai.optimalmd.com/api/v1/files/', formData, {
				headers: {
					'Content-Type': 'multipart/form-data',
					Authorization: authToken
				}
			});

			console.log('File uploaded successfully:', response.data);

			selectedFiles = selectedFiles.map((f) =>
				f.id === file.id
					? {
							...f,
							file: {
								id: response.data.id,
								collection_name: `file-${response.data.id}`,
								file: response.data,
								name: response.data.filename,
								size: response.data.meta.size,
								status: 'uploaded',
								type: 'file',
								url: `/api/v1/files/${response.data.id}`
							},
							isLoading: false,
							isUploaded: true
						}
					: f
			);

			return response.data.id;
		} catch (error) {
			console.error('Error uploading file:', error);
			selectedFiles = selectedFiles.map((f) =>
				f.id === file.id ? { ...f, isLoading: false, error: error.message } : f
			);
			return null;
		}
	};

	const uploadAllFiles = async (files) => {
		for (const file of files) {
			await uploadFile(file);
		}
	};

	const sendMessage = async (customMessage = null) => {
		const messageToSend = customMessage || message?.trim();
		if (!messageToSend || !selectedModel) return;

		if (selectedFiles.some((file) => file.isLoading)) {
			alert('Please wait for all files to finish uploading.');
			return;
		}

		// Show chat area when first message is sent
		if (!showChat) {
			showChat = true;
		}

		const messageHistory = messages
			.filter((msg) => msg?.type === 'user' || msg?.type === 'ai')
			.map((msg) => ({
				role: msg?.type === 'user' ? 'user' : 'assistant',
				content: msg?.type === 'user' ? msg?.message : msg?.response
			}));

		const payload = {
			messages: [...messageHistory, { role: 'user', content: messageToSend }],
			model: selectedModel.id,
			files: selectedFiles
				.filter((f) => f.isUploaded)
				.map((f) => ({
					type: 'file',
					collection_name: f.file.collection_name,
					id: f.file.id,
					file: f.file.file,
					name: f.file.name,
					size: f.file.size,
					status: f.file.status,
					url: f.file.url
				})),
			stream: false
		};

		console.log('payload', payload);

		try {
			loading = true;

			const userMessage = {
				type: 'user',
				message: messageToSend,
				inputPdfFiles: selectedFiles.map((f) => f.localFile.name),
				modelUsed: selectedModel ? selectedModel.name : 'Default Model'
			};

			messages = [...messages, userMessage];
			message = '';
			selectedFiles = [];

			const tempAIMessage = {
				type: 'ai',
				response: 'Preparing results',
				model: selectedModel
			};
			messages = [...messages, tempAIMessage];

			const response = await axios.post('https://ai.optimalmd.com/api/chat/completions', payload, {
				headers: {
					'Content-Type': 'application/json',
					Authorization: authToken
				}
			});

			const data = response.data;
			console.log('API Response:', data);

			messages = messages.filter((msg) => msg.response !== 'Preparing results');

			if (data.choices && data.choices[0].message.content) {
				const aiMessage = {
					type: 'ai',
					response: data.choices[0].message.content,
					modelUsed: selectedModel ? selectedModel.name : 'Default Model',
					model: selectedModel
				};

				console.log('AI Message:', data.choices[0].message.content);

				messages = [...messages, aiMessage];
			} else {
				console.error('AI response failed:', data.message);
			}
		} catch (error) {
			console.error('Error sending the message:', error);
		} finally {
			loading = false;
		}
	};

	const handleFileChange = (e) => {
		const files = Array.from(e.target.files);
		const newFiles = files.map((file) => ({
			localFile: file,
			file: null,
			isLoading: false,
			isUploaded: false,
			id: Date.now() + Math.random().toString(36).substr(2, 9)
		}));
		selectedFiles = [...selectedFiles, ...newFiles];
		uploadAllFiles(newFiles);
		e.target.files = null;
		e.target.value = null;
	};

	const uploadPdf = () => {
		fileInputRef.click();
	};

	const removePdf = (file) => {
		selectedFiles = selectedFiles.filter((f) => f.id !== file.id);
	};

	const toggleDropdown = () => {
		isDropdownOpen = !isDropdownOpen;
	};

	const selectModel = (model) => {
		selectedModel = model;
		isDropdownOpen = false;
	};

	// Start chat when user types a message and clicks send
	const startChat = () => {
		sendMessage();
	};

	const handleTextareaInput = () => {
        const textarea = inputRef;
        if (!textarea) return;
        
        // Reset height to auto to get the correct scrollHeight
        textarea.style.height = 'auto';
        
        // Calculate the required height (max 3 lines)
        const lineHeight = 20; // Approximate line height in pixels
        const maxHeight = lineHeight * 3;
        
        if (textarea.scrollHeight <= maxHeight) {
            // Expand to fit content (up to 3 lines)
            textarea.style.height = `${Math.min(textarea.scrollHeight, maxHeight)}px`;
            textarea.style.overflowY = 'hidden';
        } else {
            // Show scrollbar after 3 lines
            textarea.style.height = `${maxHeight}px`;
            textarea.style.overflowY = 'auto';
        }
    };

    // Add this to make sure it resizes when content changes
    $: if (message) {
        setTimeout(handleTextareaInput, 0);
    }

	$: suggestedQuestions = selectedModel?.info?.meta?.suggestion_prompts;
</script>

<div class="flex h-screen bg-white">
	<div class="h-full flex flex-col justify-between w-full bg-white">
		<!-- Main content area -->
		<div class="flex flex-col justify-between h-[95vh] bg-white overflow-hidden">
			<!-- Header area -->
			<div class=" justify-between items-center">
				<div class="p-2 flex items-center justify-between">
					<div class="relative" bind:this={dropdownRef}>
						<button
							on:click={toggleDropdown}
							class="flex ml-8 items-center gap-2 px-4 py-2 text-gray-700 rounded-lg border-gray-300"
							disabled={!modelsLoaded}
						>
							{#if !modelsLoaded}
								<span>Loading models...</span>
							{:else}
								<span>
									{#if selectedModel}
										<div class="flex items-center gap-2">
											<img
												src={selectedModel?.info?.meta?.profile_image_url}
												alt={selectedModel.name}
												class="w-8 h-8 rounded-full"
											/>
											<h2 class="font-semibold">
												{selectedModel.name}
											</h2>
										</div>
									{:else}
										Select Model
									{/if}
								</span>
								<ChevronDown size={16} />
							{/if}
						</button>

						{#if isDropdownOpen && modelsLoaded}
							<div
								class="absolute ml-8 w-[440px] mt-2 bg-white rounded-md shadow-lg z-10 border-gray-200"
							>
								<ul class="py-1">
									{#each publicModels as model}
										<li key={model.uniqueId} class="flex items-center hover:bg-gray-100 px-4 py-2">
											<img
												src={model?.info?.meta?.profile_image_url}
												alt={model.name}
												class="w-8 h-8 rounded-full"
											/>
											<button
												on:click={() => selectModel(model)}
												class="block w-full text-left px-4 py-2 text-sm font-semibold"
											>
												{model.name}
											</button>
										</li>
									{/each}
								</ul>
							</div>
						{/if}
					</div>

					<!-- <p class="text-xs font-bold text-gray-700">
						This tool is for informational purposes only, Consult a healthcare provider for diagnosis
						and treatment
					</p> -->

					<div></div>
				</div>
			</div>

			<!-- Chat area or welcome screen -->
			{#if !showChat}
				<!-- Welcome screen that shows initially -->
				<div class="flex-1 flex flex-col items-center justify-center p-4">
					{#if !modelsLoaded}
						<!-- Shimmer effect while models are loading -->
						<div class="avatar-shimmer shimmer"></div>
						<div class="welcome-shimmer shimmer"></div>
						<div class="welcome-shimmer shimmer" style="width: 250px; height: 60px"></div>
					{:else}
						<!-- Existing welcome screen content -->
						<div class="text-center mb-2 ">
							<div class="flex m-auto  ">
								<div class="flex m-auto  gap-2">
									<img
								src={selectedModel?.info?.meta?.profile_image_url || '/placeholder-logo.png'}
								alt="Virtual Physician"
								class="w-10 h-10 rounded-full"
								/>
								<h1 class="text-2xl font-bold text-gray-800">{selectedModel?.name}</h1>
								</div>
							</div>
							<p class="text-gray-600 text-[12px] mt-2 max-w-lg mx-auto">
								{selectedModel?.info?.meta?.description ||
									'Welcome to the AI Chat! Ask me anything about your health.'}
							</p>
							<!-- Suggested Questions Section -->
							<div class="mt-8 w-[75%] px-4 m-auto">
								<h3 class="text-sm font-semibold text-gray-500 text-left">Suggested</h3>
								<div class="relative overflow-y-scroll">
									<div class="flex flex-col gap-2 max-h-32 overflow-scroll">
										{#each suggestedQuestions as question}
											<button
												on:click={() => {
													message = question.content;
												}}
												class="w-full p-3 text-left hover:bg-gray-100 rounded-lg border-gray-200 transition-colors text-gray-700 whitespace-nowrap overflow-hidden text-ellipsis"
												title={question.content}
											>
												{question.content}
											</button>
										{/each}
									</div>
									
								</div>
							</div>
						</div>
					{/if}
				</div>
			{:else}
				<!-- Chat messages area (shown after first message) -->
				<div
					id="chatArea"
					class="flex-1 overflow-auto p-4 space-y-4 px-6 sm:px-16 md:px-32 lg:px-52 xl:px-60"
				>
					{#each messages as msg, index}
						<div key={index} class={`flex ${msg?.type === 'ai' ? 'justify-start' : 'justify-end'}`}>
							<div class="grid">
								{#if msg?.inputPdfFiles && msg?.type === 'user'}
									<div class="flex flex-wrap gap-2 mb-2">
										{#each msg?.inputPdfFiles as pdf, i}
											<div
												key={i}
												class="flex items-center space-x-1 p-1 text-primaryText bg-secondary hover:bg-secondary rounded-lg shadow-md"
											>
												<FileText size={18} class="text-red-600" />
												<span class="font-medium text-xs">
													{pdf?.length > 9 ? `${pdf?.slice(0, 5)}...${pdf?.slice(-4)}` : pdf}
												</span>
											</div>
										{/each}
									</div>
								{/if}

								{#if msg.type === 'ai'}
									<!-- Show model info above AI responses -->
									<div class="flex items-center gap-2 mb-2">
										<img
											src={msg.model?.info?.meta?.profile_image_url}
											alt={msg.modelUsed}
											class="w-6 h-6 rounded-full"
										/>
										<span class="text-xs text-gray-600 font-medium">{msg.model?.name}</span>
									</div>
								{/if}

								<div
									class={`w-full p-3 rounded-xl ${
										msg?.type === 'ai'
											? ' text-primaryText w-[90%]'
											: 'bg-gray-50 text-primaryText'
									}`}
								>
									<div class="">
										{#if msg.type === 'ai' && loading && index === messages.length - 1}
											<div class="flex gap-2">
												<Spinner /> <span class="text-gray-500">Analyzing...</span>
											</div>
										{:else if msg.type === 'ai'}
											<MarkdownRenderer markdownText={msg.response} />
											<!-- <ReactMarkdown>{msg.response}</ReactMarkdown> -->
											<!-- {msg.response} -->
										{:else}
											{msg.message}
										{/if}
									</div>
								</div>
							</div>
						</div>
					{/each}
				</div>
			{/if}

			<!-- Input area - Always visible -->
			<div bind:this={chatInputArea} class="px-6 sm:px-16 md:px-32 lg:px-52 xl:px-60 mb-4">
				<div class="flex flex-wrap gap-4 mt-4">
					{#each selectedFiles as file}
						<FileItem {file} onRemove={removePdf} isUploading={file.isLoading} />
					{/each}
				</div>
				<div
					class=" relative gap-2 flex flex-col items-center mt-2 mb-2 rounded-[30px] bg-gray-50 px-2 pb-2"
				>
					<input
						type="file"
						bind:this={fileInputRef}
						class="hidden"
						accept="application/pdf"
						multiple
						on:change={handleFileChange}
					/>
					<textarea
						bind:this={inputRef}
						rows={1}
						bind:value={message}
						on:keydown={(e) => {
							if (e.key === 'Enter' && !e.shiftKey) {
								e.preventDefault();
								sendMessage();
							}
						}}
						placeholder="How can I help you today?"
						class="bg-transparent p-3 focus:outline-none w-full text-gray-800 resize-none overflow-auto"
						style="height: auto"
					/>
					<div class="flex justify-between w-full">
						<Tooltip content="Upload Files" position="left">
							<button
								class="text-gray-600 ml-2 p-2 rounded-full border-gray-400 bg-gray-50 hover:bg-gray-100 focus:outline-none"
								on:click={uploadPdf}
							>
								<Plus size={24} />
							</button>
						</Tooltip>

						<button
							class=" mr-2 p-1 h-8 rounded-full border text-white border-gray-400 bg-black focus:outline-none"
							on:click={() => sendMessage()}
							disabled={!message?.trim() ||
								!selectedModel ||
								selectedFiles.some((file) => file.isLoading)}
						>
							<svg
								xmlns="http://www.w3.org/2000/svg"
								width="24"
								height="24"
								viewBox="0 0 24 24"
								fill="none"
								stroke="currentColor"
								stroke-width="2"
								stroke-linecap="round"
								stroke-linejoin="round"
								class="lucide lucide-arrow-up-icon lucide-arrow-up"
								><path d="m5 12 7-7 7 7" /><path d="M12 19V5" /></svg
							>
						</button>
					</div>
				</div>
			</div>
		</div>
	</div>
</div>

<style>
	.shimmer {
		position: relative;
		overflow: hidden;
		background: #f3f4f6;
	}

	.shimmer::after {
		content: '';
		position: absolute;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		background: linear-gradient(
			90deg,
			rgba(255, 255, 255, 0) 0%,
			rgba(255, 255, 255, 0.8) 50%,
			rgba(255, 255, 255, 0) 100%
		);
		animation: shimmer 1.5s infinite;
	}

	@keyframes shimmer {
		0% {
			transform: translateX(-100%);
		}
		100% {
			transform: translateX(100%);
		}
	}

	.model-selector-shimmer {
		width: 200px;
		height: 40px;
		border-radius: 8px;
	}

	.welcome-shimmer {
		width: 300px;
		height: 120px;
		border-radius: 12px;
		margin-bottom: 24px;
	}

	.avatar-shimmer {
		width: 80px;
		height: 80px;
		border-radius: 50%;
		margin: 0 auto 16px;
	}

	/* Scrollbar styling */
	.overflow-y-auto::-webkit-scrollbar {
		width: 4px;
	}

	.overflow-y-auto::-webkit-scrollbar-track {
		background: #f1f1f1;
		border-radius: 4px;
	}

	.overflow-y-auto::-webkit-scrollbar-thumb {
		background: #c1c1c1;
		border-radius: 4px;
	}

	.overflow-y-auto::-webkit-scrollbar-thumb:hover {
		background: #a1a1a1;
	}

	/* Single line question styling */
	.whitespace-nowrap {
		white-space: nowrap;
	}

	.text-ellipsis {
		text-overflow: ellipsis;
		overflow: hidden;
	}
</style>
