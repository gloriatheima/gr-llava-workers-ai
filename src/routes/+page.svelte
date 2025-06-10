<script lang="ts">
	import { onMount } from 'svelte';
	let image: File | null = null;           // 存储上传的图片文件
	let imageUrl: string | null = null;       // 图片的URL，用于预览
	let question: string = '';                // 用户输入的问题
	let isDragging: boolean = false;			// 是否正在拖拽文件
	let description: string | null = null;		// 服务器返回的回答
	let isLoading: boolean = false;				// 是否正在加载中

	const MAX_WIDTH = 800;
	const MAX_HEIGHT = 800;

	const resizeImage = (file: File): Promise<File> => {
		// 创建一个Promise来异步处理图片调整
		return new Promise((resolve) => {
			const img = new Image();
			img.src = URL.createObjectURL(file);
			img.onload = () => {
				const canvas = document.createElement('canvas');
				// 创建canvas来调整图片尺寸
				let width = img.width;
				let height = img.height;
				// 根据最大尺寸调整图片
				if (width > height) {
					if (width > MAX_WIDTH) {
						height = Math.round((height * MAX_WIDTH) / width);
						width = MAX_WIDTH;
					}
				} else {
					if (height > MAX_HEIGHT) {
						width = Math.round((width * MAX_HEIGHT) / height);
						height = MAX_HEIGHT;
					}
				}
				// 绘制调整后的图片
				canvas.width = width;
				canvas.height = height;
				const ctx = canvas.getContext('2d');
				ctx.drawImage(img, 0, 0, width, height);
				// 转换回文件对象
				canvas.toBlob((blob) => {
					if (blob) {
						resolve(new File([blob], file.name, { type: file.type }));
					}
				}, file.type);
			};
		});
	};

	const handleFileUpload = async (event: Event) => {
		const target = event.target as HTMLInputElement;
		if (target.files && target.files.length > 0) {
			image = target.files[0];
			image = await resizeImage(image);
			imageUrl = URL.createObjectURL(image);
		}
	};

	const handleDrop = async (event: DragEvent) => {
		event.preventDefault();
		isDragging = false;
		if (event.dataTransfer && event.dataTransfer.files.length > 0) {
			image = event.dataTransfer.files[0];
			image = await resizeImage(image);
			imageUrl = URL.createObjectURL(image);
		}
	};

	const handleDragOver = (event: DragEvent) => {
		event.preventDefault();
		isDragging = true;
	};

	const handleDragLeave = () => {
		isDragging = false;
	};

	const handleKeyDown = (event: KeyboardEvent) => {
		if (event.key === 'Enter' || event.key === ' ') {
			event.preventDefault();
			document.getElementById('file-input')?.click();
		}
	};

	const submitForm = async () => {
		if (!image || !question) {
			alert('Please upload an image and ask a question.');
			return;
		}

		isLoading = true;
		const formData = new FormData();
		formData.append('image', image);
		formData.append('question', question);

		try {
			// 发送请求到服务器
			const response = await fetch('/api/ask', {
				method: 'POST',
				body: formData
			});
			type Result = {
				description: string;
			};
			const result: Result = await response.json();
			description = result.description; 		// 存储服务器返回的回答
		} catch (error) {
			console.error('Error submitting form:', error);
			description = 'An error occurred while processing your request.';
		} finally {
			isLoading = false;
		}
	};
</script>

<div class="container">
	<h1>Floor is Llava</h1>
	<div
		class="upload-area {isDragging ? 'dragging' : ''}"
		role="button"
		tabindex="0"
		on:drop={handleDrop}
		on:dragover={handleDragOver}
		on:dragleave={handleDragLeave}
		on:click={() => document.getElementById('file-input')?.click()}
		on:keydown={handleKeyDown}
		aria-label="Upload Area: Drag & Drop Image or Click to Upload"
	>
		<input
			type="file"
			accept="image/*"
			on:change={handleFileUpload}
			style="display: none;"
			id="file-input"
		/>
		{#if imageUrl}
			<img src={imageUrl} alt="Upload preview" />
		{:else}
			<label for="file-input">Drag & Drop Image or Click to Upload</label>
		{/if}
	</div>
	<input
		type="text"
		class="question-input"
		placeholder="Ask a question about the photo..."
		bind:value={question}
	/>
	<button class="submit-button" on:click={submitForm}>Submit</button>
	{#if isLoading}
		<div class="loading-indicator">Processing your request...</div>
	{/if}
	{#if description}
		<div class="description">{description}</div>
	{/if}
	<div class="footer">
		<p>
  Built with <img src="./img/cloudflare.gif" alt="love" style="width: 16px; height: 16px; vertical-align: middle;"> on <a href="https://developers.cloudflare.com/workers-ai/" target="_blank"
    >Workers AI</a
  >
</p>

		<p>
			Learn more about <a
				href="https://developers.cloudflare.com/workers-ai/privacy/"
				target="_blank">Cloudflare AI data and privacy</a
			>
		</p>
		<p>
			👀 the <a
				href="https://github.com/craigsdennis/floor-is-llava-workers-ai"
				target="_blank">Workers AI code</a
			>
		</p>

	</div>
</div>

<style>
	.container {
		display: flex;
		flex-direction: column;
		align-items: center;
		margin-top: 60px;
	}

	h1 {
		color: #fe752c;
	}

	.upload-area {
		width: 400px;
		height: 300px;
		border: 3px dashed #ccc;
		display: flex;
		justify-content: center;
		align-items: center;
		margin-bottom: 30px;
		transition: background-color 0.3s;
		cursor: pointer;
		font-size: 20px;
	}

	.upload-area.dragging {
		background-color: #e0e0e0;
	}

	.upload-area img {
		max-width: 100%;
		max-height: 100%;
		display: block;
	}

	.question-input {
		width: 400px;
		padding: 15px;
		margin-bottom: 30px;
		font-size: 18px;
	}

	.submit-button {
		padding: 15px 30px;
		background-color: #fe752c;
		color: white;
		border: none;
		border-radius: 5px;
		cursor: pointer;
		font-size: 18px;
	}

	.submit-button:hover {
		background-color: #fe752c;
	}

	.loading-indicator {
		margin-top: 30px;
		font-size: 18px;
		color: #fe752c;
	}

	.description {
		margin-top: 30px;
		font-size: 20px;
		font-weight: bold;
		padding: 20px;
		border: 3px solid #fe752c;
		border-radius: 5px;
		background-color: #e0f7fa;
		color: #fe752c;
		max-width: 80%;
		text-align: center;
	}

	.footer {
		margin-top: 60px;
		text-align: center;
		font-size: 16px;
		color: #555;
	}

	.footer p {
		margin: 10px 0;
	}

	.footer a {
		color: #fe752c;
		text-decoration: none;
	}

	.footer a:hover {
		text-decoration: underline;
	}
</style>
