<!-- src/routes/+page.svelte -->
<script>
	import { onMount } from 'svelte';
	import { writable } from 'svelte/store';
	import { browser } from '$app/environment';
	
	// Stores
	const user = writable({
		name: 'Devlogue',
		level: 1,
		xp: 0,
		maxXp: 100,
		streak: 0,
		lastEntry: null,
		totalEntries: 0
	});
	
	const entries = writable([]);
	const quests = writable([]);
	const showQuests = writable(false);
	
	// State variables
	let quillEditor;
	let riveInstance;
	let journalContent = '';
	let isLoading = false;
	let showXPGain = false;
	let xpGained = 0;
	let currentUser;
	let currentEntries;
	let currentQuests;
	let showQuestsPanel;
	
	// Subscribe to stores
	user.subscribe(value => currentUser = value);
	entries.subscribe(value => currentEntries = value);
	quests.subscribe(value => currentQuests = value);
	showQuests.subscribe(value => showQuestsPanel = value);
	
	// Load Quill dynamically
	function loadQuill() {
		return new Promise((resolve, reject) => {
			// Check if Quill is already loaded
			if (window.Quill) {
				resolve(window.Quill);
				return;
			}

			// Create script element
			const script = document.createElement('script');
			script.src = 'https://cdnjs.cloudflare.com/ajax/libs/quill/1.3.7/quill.min.js';
			script.onload = () => {
				if (window.Quill) {
					resolve(window.Quill);
				} else {
					reject(new Error('Quill failed to load'));
				}
			};
			script.onerror = () => reject(new Error('Failed to load Quill script'));
			document.head.appendChild(script);
		});
	}

	// Initialize Quill editor
	onMount(async () => {
		// Only run in browser
		if (!browser) return;
		
		try {
			// Load Quill
			console.log('Loading Quill...');
			const Quill = await loadQuill();
			console.log('Quill loaded successfully:', Quill);
			
			// Wait a bit for DOM to be ready
			await new Promise(resolve => setTimeout(resolve, 100));
			
			const editorElement = document.getElementById('editor');
			if (!editorElement) {
				console.error('Editor element not found');
				return;
			}
			
			quillEditor = new Quill('#editor', {
				theme: 'snow',
				placeholder: "What's on your mind?",
				modules: {
					toolbar: [
						['bold', 'italic', 'underline'],
						['link', 'blockquote'],
						[{ 'list': 'ordered'}, { 'list': 'bullet' }],
						['clean']
					]
				}
			});
			
			quillEditor.on('text-change', () => {
				journalContent = quillEditor.root.innerHTML;
			});
			
			console.log('Quill editor initialized successfully');

		} catch (error) {
			console.error('Error loading Quill:', error);
		}
		
		// Initialize Rive avatar (independent of Quill)
		await initializeRiveAvatar();

		
		// Load saved data
		loadUserData();
		loadEntries();
		loadQuests();
	});
	
	// Initialize Rive Avatar
	async function initializeRiveAvatar() {
		// Only run in browser environment
		if (!browser) {
			console.log('Not in browser environment, skipping Rive initialization');
			return;
		}
		
		console.log('Starting Rive avatar initialization...');
		
		try {
			// Dynamic import to avoid SSR issues
			console.log('Importing Rive library...');
			const RiveModule = await import('@rive-app/canvas');
			console.log('Rive module loaded:', RiveModule);
			
			const { Rive } = RiveModule;
			
			const canvas = document.getElementById('rive-avatar');
			console.log('Canvas element found:', canvas);
			
			if (canvas) {
				console.log('Creating Rive instance with src: /avatar.riv');
				
				riveInstance = new Rive({
					src: '/avatar.riv', // Path to your .riv file in static folder
					canvas: canvas,
					autoplay: true,
                    automaticallyHandleEvents: true,
                    stateMachines: "State Machine 1", // Replace with your state machine name
					onLoad: () => {
						console.log('✅ Rive avatar loaded successfully');
						riveInstance.resizeDrawingSurfaceToCanvas();
						// Hide fallback since Rive loaded
						const fallback = document.getElementById('avatar-fallback');
						if (fallback) fallback.style.display = 'none';
					},
					onLoadError: (error) => {
						console.error('❌ Failed to load Rive avatar:', error);
						console.log('Showing fallback avatar');
						// Fallback to icon if Rive fails to load
						canvas.style.display = 'none';
						const fallback = document.getElementById('avatar-fallback');
						if (fallback) fallback.style.display = 'flex';
					}
				});
				
				console.log('Rive instance created:', riveInstance);
				
			} else {
				console.error('❌ Canvas element #rive-avatar not found');
				// Show fallback if canvas not found
				const fallback = document.getElementById('avatar-fallback');
				if (fallback) fallback.style.display = 'flex';
			}
		} catch (error) {
			console.error('❌ Failed to load Rive library:', error);
			console.log('Showing fallback due to library error');
			// Show fallback if Rive library fails to load
			const canvas = document.getElementById('rive-avatar');
			const fallback = document.getElementById('avatar-fallback');
			if (canvas) canvas.style.display = 'none';
			if (fallback) fallback.style.display = 'flex';
		}
	}
	
	// Trigger avatar animations based on user actions
	function triggerAvatarAnimation(animationName) {
		if (riveInstance) {
			// You can trigger specific animations here based on your Rive file's state machines
			// Example: riveInstance.play(animationName);
		}
	}
	
	// Save/Load functions
	function saveUserData() {
		localStorage.setItem('journal_user', JSON.stringify(currentUser));
	}
	
	function loadUserData() {
		const saved = localStorage.getItem('journal_user');
		if (saved) {
			user.set(JSON.parse(saved));
		}
	}
	
	function loadEntries() {
		const saved = localStorage.getItem('journal_entries');
		if (saved) {
			entries.set(JSON.parse(saved));
		}
	}
	
	function saveEntries() {
		localStorage.setItem('journal_entries', JSON.stringify(currentEntries));
	}
	
	function loadQuests() {
		const saved = localStorage.getItem('journal_quests');
		if (saved) {
			quests.set(JSON.parse(saved));
		}
	}
	
	function saveQuests() {
		localStorage.setItem('journal_quests', JSON.stringify(currentQuests));
	}
	
	// XP and leveling system
	function calculateXP(content) {
		const wordCount = content.replace(/<[^>]*>/g, '').split(/\s+/).filter(w => w.length > 0).length;
		return Math.min(Math.floor(wordCount / 10) * 5 + 10, 50); // 5 XP per 10 words, min 10, max 50
	}
	
	function gainXP(amount) {
		user.update(u => {
			const newXP = u.xp + amount;
			let newLevel = u.level;
			let newMaxXp = u.maxXp;
			
			if (newXP >= u.maxXp) {
				newLevel++;
				newMaxXp = u.maxXp + 50;
				showLevelUp(newLevel);
			}
			
			return { ...u, xp: newXP, level: newLevel, maxXp: newMaxXp };
		});
		
		showXPAnimation(amount);
		saveUserData();
	}
	
	function showXPAnimation(amount) {
		xpGained = amount;
		showXPGain = true;
		setTimeout(() => showXPGain = false, 2000);
	}
	
	function showLevelUp(level) {
		// Trigger celebration animation on avatar
		triggerAvatarAnimation('celebrate');
		// Simple level up notification
		alert(`🎉 Level Up! You're now level ${level}!`);
	}
	
	// Streak calculation
	function updateStreak() {
		const today = new Date().toDateString();
		const lastEntry = currentUser.lastEntry;
		
		if (lastEntry) {
			const lastDate = new Date(lastEntry).toDateString();
			const yesterday = new Date(Date.now() - 86400000).toDateString();
			
			if (lastDate === today) {
				return; // Already wrote today
			} else if (lastDate === yesterday) {
				user.update(u => ({ ...u, streak: u.streak + 1, lastEntry: today }));
			} else {
				user.update(u => ({ ...u, streak: 1, lastEntry: today }));
			}
		} else {
			user.update(u => ({ ...u, streak: 1, lastEntry: today }));
		}
		saveUserData();
	}
	
	// AI Integration (Mock Gemini API)
	async function analyzeEntry(content) {
		// Mock API call - replace with actual Gemini API
		const mockQuests = [
			"Reflect deeper on the emotions you mentioned. What triggered these feelings?",
			"Explore a specific memory related to what you wrote about today.",
			"Consider how this experience might influence your future decisions.",
			"Write about someone who has impacted this aspect of your life."
		];
		
		// Simulate API delay
		await new Promise(resolve => setTimeout(resolve, 1500));
		
		return {
			quests: mockQuests.slice(0, Math.floor(Math.random() * 3) + 1),
			themes: ['reflection', 'growth', 'relationships']
		};
	}
	
	// Journal entry submission
	async function submitEntry() {
		if (!journalContent.trim() || isLoading) return;
		
		isLoading = true;
		
		try {
			// Create entry
			const entry = {
				id: Date.now(),
				content: journalContent,
				date: new Date().toISOString(),
				wordCount: journalContent.replace(/<[^>]*>/g, '').split(/\s+/).filter(w => w.length > 0).length
			};
			
			// Add to entries
			entries.update(e => [entry, ...e]);
			saveEntries();
			
			// Update user stats
			user.update(u => ({ ...u, totalEntries: u.totalEntries + 1 }));
			updateStreak();
			
			// Calculate and gain XP
			const xpEarned = calculateXP(journalContent);
			gainXP(xpEarned);
			
			// Trigger happy animation on avatar
			triggerAvatarAnimation('happy');
			
			// Analyze entry and generate quests
			const analysis = await analyzeEntry(journalContent);
			const newQuests = analysis.quests.map(q => ({
				id: Date.now() + Math.random(),
				text: q,
				completed: false,
				xpReward: 25
			}));
			
			quests.set(newQuests);
			saveQuests();
			showQuests.set(true);
			
			// Clear editor
			quillEditor.setContents([]);
			journalContent = '';
			
		} catch (error) {
			console.error('Error submitting entry:', error);
		} finally {
			isLoading = false;
		}
	}
	
	function completeQuest(questId) {
		quests.update(q => q.map(quest => 
			quest.id === questId ? { ...quest, completed: true } : quest
		));
		gainXP(25);
		saveQuests();
	}
	
	function closeQuests() {
		showQuests.set(false);
	}
</script>

<svelte:head>
	<title>Devlogue</title>
	<link href="https://cdnjs.cloudflare.com/ajax/libs/quill/1.3.7/quill.snow.css" rel="stylesheet">
	<link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css" rel="stylesheet">
</svelte:head>

<div class="app">
	<!-- Header -->
	<header class="header">
		<div class="header-content">
			<div class="user-info">
				<div class="avatar-container">
					<!-- Rive avatar canvas -->
					<canvas id="rive-avatar" class="rive-canvas" width="60" height="60"></canvas>
					<!-- Fallback icon if Rive fails to load -->
					<div id="avatar-fallback" class="avatar">
						<i class="fas fa-user-astronaut"></i>
					</div>
				</div>
				<div class="user-details">
					<h2>{currentUser.name}</h2>
					<p>Level {currentUser.level} Devloguer</p>
				</div>
			</div>
			
			<div class="stats">
				<div class="stat">
					<i class="fas fa-fire"></i>
					<span>{currentUser.streak} day streak</span>
				</div>
				<div class="stat">
					<i class="fas fa-book"></i>
					<span>{currentUser.totalEntries} entries</span>
				</div>
			</div>
		</div>
		
		<!-- XP Bar -->
		<div class="xp-container">
			<div class="xp-bar">
				<div class="xp-fill" style="width: {(currentUser.xp / currentUser.maxXp) * 100}%"></div>
				<span class="xp-text">{currentUser.xp} / {currentUser.maxXp} XP</span>
			</div>
			{#if showXPGain}
				<div class="xp-gain">+{xpGained} XP</div>
			{/if}
		</div>
	</header>

	<!-- Main Content -->
	<main class="main">
		<div class="journal-section">
			<div class="editor-container">
				<div id="editor"></div>
				<div class="editor-footer">
					<div class="word-count">
						Words: {journalContent.replace(/<[^>]*>/g, '').split(/\s+/).filter(w => w.length > 0).length}
					</div>
					<button 
						class="submit-btn" 
						class:loading={isLoading}
						on:click={submitEntry}
						disabled={isLoading || !journalContent.trim()}
					>
						{#if isLoading}
							<i class="fas fa-spinner fa-spin"></i>
							Analyzing...
						{:else}
							<i class="fas fa-feather"></i>
							Submit Entry
						{/if}
					</button>
				</div>
			</div>
		</div>

		<!-- Previous Entries -->
		{#if currentEntries.length > 0}
			<div class="entries-section">
				<h3><i class="fas fa-history"></i> Recent Entries</h3>
				<div class="entries-list">
					{#each currentEntries.slice(0, 3) as entry}
						<div class="entry-card">
							<div class="entry-date">
								{new Date(entry.date).toLocaleDateString()}
							</div>
							<div class="entry-preview">
								{@html entry.content.substring(0, 150)}...
							</div>
							<div class="entry-stats">
								<span><i class="fas fa-pen"></i> {entry.wordCount} words</span>
							</div>
						</div>
					{/each}
				</div>
			</div>
		{/if}
	</main>

	<!-- Quests Panel -->
	{#if showQuestsPanel}
		<div class="quests-overlay" on:click={closeQuests}>
			<div class="quests-panel" on:click|stopPropagation>
				<div class="quests-header">
					<h3><i class="fas fa-scroll"></i> New Quests Available!</h3>
					<button class="close-btn" on:click={closeQuests}>
						<i class="fas fa-times"></i>
					</button>
				</div>
				<div class="quests-content">
					<p>Your entry has unlocked new reflection quests. Complete them to earn bonus XP!</p>
					<div class="quests-list">
						{#each currentQuests as quest}
							<div class="quest-card" class:completed={quest.completed}>
								<div class="quest-text">{quest.text}</div>
								<div class="quest-reward">
									<span><i class="fas fa-star"></i> {quest.xpReward} XP</span>
									{#if !quest.completed}
										<button class="complete-quest-btn" on:click={() => completeQuest(quest.id)}>
											Complete
										</button>
									{:else}
										<span class="completed-badge">✓ Done</span>
									{/if}
								</div>
							</div>
						{/each}
					</div>
				</div>
			</div>
		</div>
	{/if}
</div>

<style>
	:global(body) {
		margin: 0;
		font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		min-height: 100vh;
	}

	.app {
		min-height: 100vh;
		background: rgba(255, 255, 255, 0.1);
		backdrop-filter: blur(10px);
	}

	.header {
		background: rgba(255, 255, 255, 0.95);
		backdrop-filter: blur(20px);
		border-bottom: 1px solid rgba(255, 255, 255, 0.2);
		padding: 1.5rem 2rem;
		box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
	}

	.header-content {
		display: flex;
		justify-content: space-between;
		align-items: center;
		max-width: 1200px;
		margin: 0 auto;
		margin-bottom: 1rem;
	}

	.user-info {
		display: flex;
		align-items: center;
		gap: 1rem;
	}

	.avatar-container {
		position: relative;
		width: 120px;
		height: 120px;
	}

	.rive-canvas {
		position: absolute;
		top: 0;
		left: 0;
		width: 120px;
		height: 120px;
		border-radius: 50%;
		box-shadow: 0 8px 32px rgba(102, 126, 234, 0.3);
		background: linear-gradient(135deg, #667eea, #764ba2);
		z-index: 2;
	}

	.avatar {
		position: absolute;
		top: 0;
		left: 0;
		width: 120px;
		height: 120px;
		border-radius: 50%;
		background: linear-gradient(135deg, #667eea, #764ba2);
		display: flex;
		align-items: center;
		justify-content: center;
		color: white;
		font-size: 3rem;
		box-shadow: 0 8px 32px rgba(102, 126, 234, 0.3);
		z-index: 1;
	}

	.user-details h2 {
		margin: 0;
		color: #333;
		font-size: 1.3rem;
	}

	.user-details p {
		margin: 0;
		color: #666;
		font-size: 0.9rem;
	}

    .user-info {
        gap: 2rem; /* Increased gap for balance */
    }

	.stats {
		display: flex;
		gap: 2rem;
	}

	.stat {
		display: flex;
		align-items: center;
		gap: 0.5rem;
		color: #555;
		font-weight: 500;
	}

	.stat i {
		color: #667eea;
	}

	.xp-container {
		position: relative;
		max-width: 1200px;
		margin: 0 auto;
	}

	.xp-bar {
		background: rgba(0, 0, 0, 0.1);
		border-radius: 25px;
		height: 20px;
		position: relative;
		overflow: hidden;
	}

	.xp-fill {
		background: linear-gradient(90deg, #667eea, #764ba2);
		height: 100%;
		border-radius: 25px;
		transition: width 0.5s ease;
	}

	.xp-text {
		position: absolute;
		top: 50%;
		left: 50%;
		transform: translate(-50%, -50%);
		color: #333;
		font-size: 0.8rem;
		font-weight: 600;
	}

	.xp-gain {
		position: absolute;
		top: -30px;
		right: 0;
		background: #4CAF50;
		color: white;
		padding: 0.3rem 0.8rem;
		border-radius: 15px;
		font-size: 0.8rem;
		font-weight: 600;
		animation: slideUp 2s ease-out;
	}

	@keyframes slideUp {
		0% { opacity: 0; transform: translateY(20px); }
		20% { opacity: 1; transform: translateY(0); }
		80% { opacity: 1; transform: translateY(0); }
		100% { opacity: 0; transform: translateY(-20px); }
	}

	.main {
		max-width: 1200px;
		margin: 0 auto;
		padding: 2rem;
		display: grid;
		grid-template-columns: 2fr 1fr;
		gap: 2rem;
	}

	.journal-section {
		background: rgba(255, 255, 255, 0.95);
		backdrop-filter: blur(20px);
		border-radius: 20px;
		box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
		overflow: hidden;
	}

	.editor-container {
		position: relative;
		display: flex;
		flex-direction: column;
        min-height: 0;
	}

	:global(#editor) {
		flex: 1 1 auto;
		border: none !important;
        min-height: 600px;
        overflow-y: auto;
	}

	:global(.ql-toolbar) {
		border-bottom: 1px solid #eee !important;
		border-top: none !important;
		border-left: none !important;
		border-right: none !important;
	}

	:global(.ql-container) {
		border: none !important;
		font-size: 16px;
	}

	.editor-footer {
        flex-shrink: 0;
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 1rem 1.5rem;
		background: rgba(0, 0, 0, 0.02);
		border-top: 1px solid #eee;
	}

	.word-count {
		color: #666;
		font-size: 0.9rem;
	}

	.submit-btn {
		background: linear-gradient(135deg, #667eea, #764ba2);
		color: white;
		border: none;
		padding: 0.8rem 1.5rem;
		border-radius: 25px;
		font-weight: 600;
		cursor: pointer;
		transition: all 0.3s ease;
		display: flex;
		align-items: center;
		gap: 0.5rem;
	}

	.submit-btn:hover:not(:disabled) {
		transform: translateY(-2px);
		box-shadow: 0 8px 20px rgba(102, 126, 234, 0.3);
	}

	.submit-btn:disabled {
		opacity: 0.6;
		cursor: not-allowed;
	}

	.submit-btn.loading {
		pointer-events: none;
	}

	.entries-section {
		background: rgba(255, 255, 255, 0.9);
		backdrop-filter: blur(20px);
		border-radius: 20px;
		padding: 1.5rem;
		box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
		height: fit-content;
	}

	.entries-section h3 {
		margin: 0 0 1rem 0;
		color: #333;
		display: flex;
		align-items: center;
		gap: 0.5rem;
	}

	.entries-list {
		display: flex;
		flex-direction: column;
		gap: 1rem;
	}

	.entry-card {
		background: rgba(255, 255, 255, 0.5);
		border-radius: 12px;
		padding: 1rem;
		border: 1px solid rgba(255, 255, 255, 0.3);
	}

	.entry-date {
		color: #667eea;
		font-size: 0.8rem;
		font-weight: 600;
		margin-bottom: 0.5rem;
	}

	.entry-preview {
		color: #555;
		font-size: 0.9rem;
		line-height: 1.4;
		margin-bottom: 0.5rem;
	}

	.entry-stats {
		color: #888;
		font-size: 0.8rem;
	}

	.quests-overlay {
		position: fixed;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		background: rgba(0, 0, 0, 0.5);
		backdrop-filter: blur(5px);
		display: flex;
		align-items: center;
		justify-content: center;
		z-index: 1000;
		animation: fadeIn 0.3s ease;
	}

	@keyframes fadeIn {
		from { opacity: 0; }
		to { opacity: 1; }
	}

	.quests-panel {
		background: white;
		border-radius: 20px;
		max-width: 600px;
		width: 90%;
		max-height: 80vh;
		overflow-y: auto;
		box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
		animation: slideIn 0.3s ease;
	}

	@keyframes slideIn {
		from { transform: translateY(-50px); opacity: 0; }
		to { transform: translateY(0); opacity: 1; }
	}

	.quests-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 1.5rem;
		border-bottom: 1px solid #eee;
	}

	.quests-header h3 {
		margin: 0;
		color: #333;
		display: flex;
		align-items: center;
		gap: 0.5rem;
	}

	.close-btn {
		background: none;
		border: none;
		font-size: 1.2rem;
		cursor: pointer;
		color: #666;
		padding: 0.5rem;
		border-radius: 50%;
		transition: background 0.3s ease;
	}

	.close-btn:hover {
		background: rgba(0, 0, 0, 0.1);
	}

	.quests-content {
		padding: 1.5rem;
	}

	.quests-content p {
		color: #666;
		margin-bottom: 1.5rem;
	}

	.quests-list {
		display: flex;
		flex-direction: column;
		gap: 1rem;
	}

	.quest-card {
		background: rgba(102, 126, 234, 0.05);
		border: 1px solid rgba(102, 126, 234, 0.2);
		border-radius: 12px;
		padding: 1.5rem;
		transition: all 0.3s ease;
	}

	.quest-card.completed {
		background: rgba(76, 175, 80, 0.05);
		border-color: rgba(76, 175, 80, 0.2);
	}

	.quest-text {
		color: #333;
		margin-bottom: 1rem;
		line-height: 1.5;
	}

	.quest-reward {
		display: flex;
		justify-content: space-between;
		align-items: center;
	}

	.quest-reward span {
		color: #667eea;
		font-weight: 600;
		display: flex;
		align-items: center;
		gap: 0.3rem;
	}

	.complete-quest-btn {
		background: #4CAF50;
		color: white;
		border: none;
		padding: 0.5rem 1rem;
		border-radius: 20px;
		font-weight: 600;
		cursor: pointer;
		transition: all 0.3s ease;
	}

	.complete-quest-btn:hover {
		background: #45a049;
		transform: translateY(-1px);
	}

	.completed-badge {
		color: #4CAF50;
		font-weight: 600;
	}

	@media (max-width: 768px) {
		.main {
			grid-template-columns: 1fr;
			padding: 1rem;
		}
		
		.header {
			padding: 1rem;
		}
		
		.header-content {
			flex-direction: column;
			gap: 1rem;
		}
		
		.stats {
			gap: 1rem;
		}
	}
</style>