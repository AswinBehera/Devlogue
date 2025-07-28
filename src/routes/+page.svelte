<!-- src/routes/+page.svelte -->
<script>
    import { onMount } from "svelte";
    import { writable } from "svelte/store";
    import { browser } from "$app/environment";

    // Stores
    const user = writable({
        name: "Devlogue",
        level: 1,
        xp: 0,
        maxXp: 100,
        streak: 0,
        lastEntry: null,
        totalEntries: 0,
    });

    const entries = writable([]);
    const activeQuests = writable([]);
    const completedQuests = writable([]);
    const showQuests = writable(false);

    // State variables
    let quillEditor;
    let riveInstance;
    let journalContent = "";
    let isLoading = false;
    let showXPGain = false;
    let xpGained = 0;
    let currentUser;
    let currentEntries;
    let currentActiveQuests;
    let currentCompletedQuests;
    let showQuestsPanel;

    // Quest-related state
    let currentQuest = null;
    let isQuestMode = false;
    let isEvaluatingQuest = false;
    let questEvaluation = null;
    let showEvaluationModal = false;
    let activeQuestTab = 'active'; // 'active' or 'completed'

    // Subscribe to stores
    user.subscribe((value) => (currentUser = value));
    entries.subscribe((value) => (currentEntries = value));
    activeQuests.subscribe((value) => (currentActiveQuests = value));
    completedQuests.subscribe((value) => (currentCompletedQuests = value));
    showQuests.subscribe((value) => (showQuestsPanel = value));

    // Load Quill dynamically
    function loadQuill() {
        return new Promise((resolve, reject) => {
            if (window.Quill) {
                resolve(window.Quill);
                return;
            }

            const script = document.createElement("script");
            script.src = "https://cdnjs.cloudflare.com/ajax/libs/quill/1.3.7/quill.min.js";
            script.onload = () => {
                if (window.Quill) {
                    resolve(window.Quill);
                } else {
                    reject(new Error("Quill failed to load"));
                }
            };
            script.onerror = () => reject(new Error("Failed to load Quill script"));
            document.head.appendChild(script);
        });
    }

    // Initialize Quill editor
    onMount(async () => {
        if (!browser) return;

        try {
            console.log("Loading Quill...");
            const Quill = await loadQuill();
            console.log("Quill loaded successfully:", Quill);

            await new Promise((resolve) => setTimeout(resolve, 100));

            const editorElement = document.getElementById("editor");
            if (!editorElement) {
                console.error("Editor element not found");
                return;
            }

            quillEditor = new Quill("#editor", {
                theme: "snow",
                placeholder: "What's on your mind?",
                modules: {
                    toolbar: [
                        ["bold", "italic", "underline"],
                        ["link", "blockquote"],
                        [{ list: "ordered" }, { list: "bullet" }],
                        ["clean"],
                    ],
                },
            });

            quillEditor.on("text-change", () => {
                journalContent = quillEditor.root.innerHTML;
            });

            console.log("Quill editor initialized successfully");
        } catch (error) {
            console.error("Error loading Quill:", error);
        }

        await initializeRiveAvatar();
        loadUserData();
        loadEntries();
        loadQuests();
    });

    // Initialize Rive Avatar
    async function initializeRiveAvatar() {
        if (!browser) {
            console.log("Not in browser environment, skipping Rive initialization");
            return;
        }

        console.log("Starting Rive avatar initialization...");

        try {
            console.log("Importing Rive library...");
            const RiveModule = await import("@rive-app/canvas");
            console.log("Rive module loaded:", RiveModule);

            const { Rive } = RiveModule;

            const canvas = document.getElementById("rive-avatar");
            console.log("Canvas element found:", canvas);

            if (canvas) {
                console.log("Creating Rive instance with src: /avatar.riv");

                riveInstance = new Rive({
                    src: "/avatar.riv",
                    canvas: canvas,
                    autoplay: true,
                    automaticallyHandleEvents: true,
                    stateMachines: "State Machine 1",
                    onLoad: () => {
                        console.log("✅ Rive avatar loaded successfully");
                        riveInstance.resizeDrawingSurfaceToCanvas();
                        const fallback = document.getElementById("avatar-fallback");
                        if (fallback) fallback.style.display = "none";
                    },
                    onLoadError: (error) => {
                        console.error("❌ Failed to load Rive avatar:", error);
                        console.log("Showing fallback avatar");
                        canvas.style.display = "none";
                        const fallback = document.getElementById("avatar-fallback");
                        if (fallback) fallback.style.display = "flex";
                    },
                });

                console.log("Rive instance created:", riveInstance);
            } else {
                console.error("❌ Canvas element #rive-avatar not found");
                const fallback = document.getElementById("avatar-fallback");
                if (fallback) fallback.style.display = "flex";
            }
        } catch (error) {
            console.error("❌ Failed to load Rive library:", error);
            console.log("Showing fallback due to library error");
            const canvas = document.getElementById("rive-avatar");
            const fallback = document.getElementById("avatar-fallback");
            if (canvas) canvas.style.display = "none";
            if (fallback) fallback.style.display = "flex";
        }
    }

    function triggerAvatarAnimation(animationName) {
        if (riveInstance) {
            // Trigger specific animations based on your Rive file's state machines
        }
    }

    // Save/Load functions
    function saveUserData() {
        if (browser) {
            localStorage.setItem("journal_user", JSON.stringify(currentUser));
        }
    }

    function loadUserData() {
        if (browser) {
            const saved = localStorage.getItem("journal_user");
            if (saved) {
                user.set(JSON.parse(saved));
            }
        }
    }

    function loadEntries() {
        if (browser) {
            const saved = localStorage.getItem("journal_entries");
            if (saved) {
                entries.set(JSON.parse(saved));
            }
        }
    }

    function saveEntries() {
        if (browser) {
            localStorage.setItem("journal_entries", JSON.stringify(currentEntries));
        }
    }

    function loadQuests() {
        if (browser) {
            const savedActive = localStorage.getItem("journal_active_quests");
            const savedCompleted = localStorage.getItem("journal_completed_quests");
            if (savedActive) {
                activeQuests.set(JSON.parse(savedActive));
            }
            if (savedCompleted) {
                completedQuests.set(JSON.parse(savedCompleted));
            }
        }
    }

    function saveQuests() {
        if (browser) {
            localStorage.setItem("journal_active_quests", JSON.stringify(currentActiveQuests));
            localStorage.setItem("journal_completed_quests", JSON.stringify(currentCompletedQuests));
        }
    }

    // XP and leveling system
    function calculateXP(content) {
        const wordCount = content
            .replace(/<[^>]*>/g, "")
            .split(/\s+/)
            .filter((w) => w.length > 0).length;
        return Math.min(Math.floor(wordCount / 10) * 5 + 10, 50);
    }

    function gainXP(amount) {
        user.update((u) => {
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
        setTimeout(() => (showXPGain = false), 2000);
    }

    function showLevelUp(level) {
        triggerAvatarAnimation("celebrate");
        alert(`🎉 Level Up! You're now level ${level}!`);
    }

    function updateStreak() {
        const today = new Date().toDateString();
        const lastEntry = currentUser.lastEntry;

        if (lastEntry) {
            const lastDate = new Date(lastEntry).toDateString();
            const yesterday = new Date(Date.now() - 86400000).toDateString();

            if (lastDate === today) {
                return;
            } else if (lastDate === yesterday) {
                user.update((u) => ({
                    ...u,
                    streak: u.streak + 1,
                    lastEntry: today,
                }));
            } else {
                user.update((u) => ({ ...u, streak: 1, lastEntry: today }));
            }
        } else {
            user.update((u) => ({ ...u, streak: 1, lastEntry: today }));
        }
        saveUserData();
    }

    // AI Integration for entry analysis
    async function analyzeEntry(content) {
        const prompt = `
You are a journaling coach. Read the following journal entry and:
1. Suggest 2-3 personalized reflection quests (as short prompts).
2. List 2-3 themes (single words) that summarize the entry.

Journal Entry:
${content}

Respond in JSON:
{
  "quests": ["quest1", "quest2", ...],
  "themes": ["theme1", "theme2", ...]
}
`;

        try {
            const response = await fetch("http://localhost:11434/api/generate", {
                method: "POST",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify({
                    model: "deepseek-r1",
                    prompt,
                    stream: false,
                }),
            });

            const data = await response.json();
            const text = data.response;

            let result;
            try {
                const match = text.match(/\{[\s\S]*\}/);
                result = match ? JSON.parse(match[0]) : JSON.parse(text);
            } catch (e) {
                console.error("Failed to parse Ollama response:", text);
                throw e;
            }

            return result;
        } catch (error) {
            console.error("Error calling Ollama API:", error);
            // Fallback to mock data if API fails
            return {
                quests: [
                    "Reflect deeper on the emotions you mentioned. What triggered these feelings?",
                    "Consider how this experience might influence your future decisions.",
                    "Write about someone who has impacted this aspect of your life."
                ],
                themes: ["reflection", "growth", "introspection"]
            };
        }
    }

    // AI Integration for quest evaluation
    async function evaluateQuestResponse(questText, response) {
        const prompt = `
You are an AI coach evaluating a journal response to a reflection quest. 

Quest: "${questText}"
Response: "${response}"

Evaluate the response on:
1. Relevance to the quest (0-10)
2. Depth of reflection (0-10) 
3. Authenticity and personal insight (0-10)
4. Watch for meaningful short phrases that might be profound

Provide constructive feedback and assign XP (10-50 based on quality).

Respond in JSON:
{
  "relevance_score": 8,
  "depth_score": 7,
  "authenticity_score": 9,
  "total_score": 24,
  "xp_awarded": 35,
  "feedback": "Your response shows great self-awareness...",
  "strengths": ["honest reflection", "specific examples"],
  "areas_for_growth": ["could explore emotions deeper"]
}
`;

        try {
            const response_api = await fetch("http://localhost:11434/api/generate", {
                method: "POST",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify({
                    model: "deepseek-r1",
                    prompt,
                    stream: false,
                }),
            });

            const data = await response_api.json();
            const text = data.response;

            let result;
            try {
                const match = text.match(/\{[\s\S]*\}/);
                result = match ? JSON.parse(match[0]) : JSON.parse(text);
            } catch (e) {
                console.error("Failed to parse quest evaluation:", text);
                throw e;
            }

            return result;
        } catch (error) {
            console.error("Error evaluating quest:", error);
            // Fallback evaluation
            const wordCount = response.replace(/<[^>]*>/g, '').split(/\s+/).filter(w => w.length > 0).length;
            const baseXP = Math.min(Math.max(wordCount * 2, 15), 45);
            
            return {
                relevance_score: 7,
                depth_score: 6,
                authenticity_score: 7,
                total_score: 20,
                xp_awarded: baseXP,
                feedback: "Thank you for your thoughtful response. Keep exploring your thoughts and feelings.",
                strengths: ["personal reflection"],
                areas_for_growth: ["continue developing self-awareness"]
            };
        }
    }

    // Quest management functions
    function startQuestResponse(quest) {
        currentQuest = quest;
        isQuestMode = true;
        
        // Update editor placeholder and clear content
        if (quillEditor) {
            quillEditor.setContents([{ insert: '\n' }]);
            journalContent = "";
            
            // Update placeholder (this is a bit hacky but works)
            const placeholder = document.querySelector('.ql-editor[data-placeholder]');
            if (placeholder) {
                placeholder.setAttribute('data-placeholder', `Reflecting on: "${quest.text}"`);
            }
        }
        
        // Close quests panel
        showQuests.set(false);
    }

    function cancelQuestResponse() {
        currentQuest = null;
        isQuestMode = false;
        
        // Reset placeholder
        if (quillEditor) {
            quillEditor.setContents([{ insert: '\n' }]);
            journalContent = "";
            
            const placeholder = document.querySelector('.ql-editor[data-placeholder]');
            if (placeholder) {
                placeholder.setAttribute('data-placeholder', "What's on your mind?");
            }
        }
    }

    // Enhanced submit function
    async function submitEntry() {
        if (!journalContent.trim() || isLoading) return;

        isLoading = true;

        try {
            if (isQuestMode && currentQuest) {
                // Quest response submission
                isEvaluatingQuest = true;
                
                // Evaluate the quest response
                const evaluation = await evaluateQuestResponse(currentQuest.text, journalContent);
                
                // Create quest response entry
                const questEntry = {
                    id: Date.now(),
                    content: journalContent,
                    date: new Date().toISOString(),
                    wordCount: journalContent.replace(/<[^>]*>/g, "").split(/\s+/).filter((w) => w.length > 0).length,
                    type: 'quest_response',
                    questId: currentQuest.id,
                    questText: currentQuest.text,
                    evaluation: evaluation
                };

                // Add to entries
                entries.update((e) => [questEntry, ...e]);
                saveEntries();

                // Move quest from active to completed
                activeQuests.update(quests => quests.filter(q => q.id !== currentQuest.id));
                completedQuests.update(completed => [{
                    ...currentQuest,
                    completed: true,
                    completedDate: new Date().toISOString(),
                    response: journalContent,
                    evaluation: evaluation
                }, ...completed]);
                saveQuests();

                // Award XP based on evaluation
                gainXP(evaluation.xp_awarded);
                
                // Update user stats
                user.update((u) => ({ ...u, totalEntries: u.totalEntries + 1 }));
                updateStreak();

                // Show evaluation modal
                questEvaluation = evaluation;
                showEvaluationModal = true;
                isEvaluatingQuest = false;

                // Trigger avatar animation
                triggerAvatarAnimation("happy");

                // Reset quest mode
                currentQuest = null;
                isQuestMode = false;

                // Reset placeholder
                const placeholder = document.querySelector('.ql-editor[data-placeholder]');
                if (placeholder) {
                    placeholder.setAttribute('data-placeholder', "What's on your mind?");
                }

            } else {
                // Regular journal entry submission
                const entry = {
                    id: Date.now(),
                    content: journalContent,
                    date: new Date().toISOString(),
                    wordCount: journalContent.replace(/<[^>]*>/g, "").split(/\s+/).filter((w) => w.length > 0).length,
                    type: 'journal_entry'
                };

                entries.update((e) => [entry, ...e]);
                saveEntries();

                user.update((u) => ({ ...u, totalEntries: u.totalEntries + 1 }));
                updateStreak();

                const xpEarned = calculateXP(journalContent);
                gainXP(xpEarned);

                triggerAvatarAnimation("happy");

                // Analyze entry and generate new quests
                const analysis = await analyzeEntry(journalContent);
                const newQuests = analysis.quests.map((q) => ({
                    id: Date.now() + Math.random(),
                    text: q,
                    created: new Date().toISOString(),
                    fromEntry: entry.id
                }));

                activeQuests.update(existing => [...existing, ...newQuests]);
                saveQuests();
                showQuests.set(true);
            }

            // Clear editor
            quillEditor.setContents([{ insert: '\n' }]);
            journalContent = "";

        } catch (error) {
            console.error("Error submitting entry:", error);
            isEvaluatingQuest = false;
        } finally {
            isLoading = false;
        }
    }

    function closeEvaluationModal() {
        showEvaluationModal = false;
        questEvaluation = null;
    }

    function closeQuests() {
        showQuests.set(false);
    }

    function switchQuestTab(tab) {
        activeQuestTab = tab;
    }

    // Get button text and style based on current mode
    $: submitButtonText = isQuestMode 
        ? (isEvaluatingQuest ? "Evaluating Quest..." : "Submit Quest Response")
        : (isLoading ? "Analyzing..." : "Submit Entry");

    $: submitButtonIcon = isQuestMode 
        ? (isEvaluatingQuest ? "fa-spinner fa-spin" : "fa-scroll")
        : (isLoading ? "fa-spinner fa-spin" : "fa-feather");

    $: editorPlaceholder = isQuestMode && currentQuest 
        ? `Reflecting on: "${currentQuest.text}"`
        : "What's on your mind?";
</script>

<svelte:head>
    <title>Devlogue</title>
    <link href="https://cdnjs.cloudflare.com/ajax/libs/quill/1.3.7/quill.snow.css" rel="stylesheet" />
    <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css" rel="stylesheet" />
</svelte:head>

<div class="app">
    <!-- Header -->
    <header class="header">
        <div class="header-content">
            <div class="user-info">
                <div class="avatar-container">
                    <canvas id="rive-avatar" class="rive-canvas" width="120" height="120"></canvas>
                    <div id="avatar-fallback" class="avatar">
                        <i class="fas fa-user-astronaut"></i>
                    </div>
                </div>
                <div class="user-details">
                    <h2>{currentUser.name}</h2>
                    <p>Level {currentUser.level} Devloguer</p>
                    {#if isQuestMode && currentQuest}
                        <p class="quest-indicator">
                            <i class="fas fa-scroll"></i> 
                            Quest Mode Active
                        </p>
                    {/if}
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
                <div class="stat">
                    <i class="fas fa-scroll"></i>
                    <span>{currentActiveQuests.length} active quests</span>
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
            {#if isQuestMode && currentQuest}
                <div class="quest-banner">
                    <div class="quest-banner-content">
                        <h4><i class="fas fa-scroll"></i> Quest Response</h4>
                        <p>"{currentQuest.text}"</p>
                        <button class="cancel-quest-btn" on:click={cancelQuestResponse}>
                            <i class="fas fa-times"></i> Cancel
                        </button>
                    </div>
                </div>
            {/if}
            
            <div class="editor-container">
                <div id="editor"></div>
                <div class="editor-footer">
                    <div class="word-count">
                        Words: {journalContent.replace(/<[^>]*>/g, "").split(/\s+/).filter((w) => w.length > 0).length}
                    </div>
                    <button 
                        class="submit-btn" 
                        class:loading={isLoading || isEvaluatingQuest}
                        class:quest-mode={isQuestMode}
                        on:click={submitEntry}
                        disabled={isLoading || isEvaluatingQuest || !journalContent.trim()}
                    >
                        <i class="fas {submitButtonIcon}"></i>
                        {submitButtonText}
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
                        <div class="entry-card" class:quest-entry={entry.type === 'quest_response'}>
                            <div class="entry-header">
                                <div class="entry-date">
                                    {new Date(entry.date).toLocaleDateString()}
                                </div>
                                {#if entry.type === 'quest_response'}
                                    <div class="entry-type">
                                        <i class="fas fa-scroll"></i> Quest Response
                                    </div>
                                {/if}
                            </div>
                            {#if entry.type === 'quest_response' && entry.questText}
                                <div class="quest-context">
                                    Quest: {entry.questText}
                                </div>
                            {/if}
                            <div class="entry-preview">
                                {@html entry.content.substring(0, 150)}...
                            </div>
                            <div class="entry-stats">
                                <span><i class="fas fa-pen"></i> {entry.wordCount} words</span>
                                {#if entry.evaluation}
                                    <span><i class="fas fa-star"></i> {entry.evaluation.xp_awarded} XP earned</span>
                                {/if}
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
                    <h3><i class="fas fa-scroll"></i> Quest Hub</h3>
                    <button class="close-btn" on:click={closeQuests}>
                        <i class="fas fa-times"></i>
                    </button>
                </div>
                
                <div class="quest-tabs">
                    <button 
                        class="tab-btn" 
                        class:active={activeQuestTab === 'active'}
                        on:click={() => switchQuestTab('active')}
                    >
                        Active Quests ({currentActiveQuests.length})
                    </button>
                    <button 
                        class="tab-btn" 
                        class:active={activeQuestTab === 'completed'}
                        on:click={() => switchQuestTab('completed')}
                    >
                        Completed ({currentCompletedQuests.length})
                    </button>
                </div>

                <div class="quests-content">
                    {#if activeQuestTab === 'active'}
                        {#if currentActiveQuests.length > 0}
                            <p>Complete these quests by writing thoughtful journal responses. Your responses will be evaluated by AI for depth and relevance.</p>
                            <div class="quests-list">
                                {#each currentActiveQuests as quest}
                                    <div class="quest-card">
                                        <div class="quest-text">{quest.text}</div>
                                        <div class="quest-actions">
                                            <span class="quest-reward">
                                                <i class="fas fa-star"></i> Variable XP
                                            </span>
                                            <button 
                                                class="start-quest-btn" 
                                                on:click={() => startQuestResponse(quest)}
                                            >
                                                <i class="fas fa-pen"></i> Write Response
                                            </button>
                                        </div>
                                    </div>
                                {/each}
                            </div>
                        {:else}
                            <div class="empty-state">
                                <i class="fas fa-scroll"></i>
                                <p>No active quests. Write a journal entry to unlock new reflection quests!</p>
                            </div>
                        {/if}
                    {:else}
                        {#if currentCompletedQuests.length > 0}
                            <div class="quests-list">
                                {#each currentCompletedQuests as quest}
                                    <div class="quest-card completed">
                                        <div class="quest-text">{quest.text}</div>
                                        <div class="quest-completion">
                                            <div class="completion-info">
                                                <span class="completed-badge">
                                                    <i class="fas fa-check"></i> 
                                                    Completed {new Date(quest.completedDate).toLocaleDateString()}
                                                </span>
                                                {#if quest.evaluation}
                                                    <span class="xp-earned">
                                                        <i class="fas fa-star"></i> 
                                                        {quest.evaluation.xp_awarded} XP
                                                    </span>
                                                {/if}
                                            </div>
                                        </div>
                                    </div>
                                {/each}
                            </div>
                        {:else}
                            <div class="empty-state">
                                <i class="fas fa-trophy"></i>
                                <p>No completed quests yet. Start your reflection journey!</p>
                            </div>
                        {/if}
                    {/if}
                </div>
            </div>
        </div>
    {/if}

    <!-- Quest Evaluation Modal -->
    {#if showEvaluationModal && questEvaluation}
        <div class="modal-overlay" on:click={closeEvaluationModal}>
            <div class="evaluation-modal" on:click|stopPropagation>
                <div class="modal-header">
                    <h3><i class="fas fa-star"></i> Quest Evaluation</h3>
                    <button class="close-btn" on:click={closeEvaluationModal}>
                        <i class="fas fa-times"></i>
                    </button>
                </div>
                
                <div class="modal-content">
                    <div class="score-summary">
                        <div class="xp-awarded">
                            <i class="fas fa-star"></i>
                            <span>{questEvaluation.xp_awarded} XP Awarded!</span>
                        </div>
                        
                        <div class="score-breakdown">
                                                        <div class="score-item">
                                <span>Relevance:</span>
                                <div class="score-bar">
                                    <div class="score-fill" style="width: {questEvaluation.relevance_score * 10}%"></div>
                                    <span>{questEvaluation.relevance_score}/10</span>
                                </div>
                            </div>
                            <div class="score-item">
                                <span>Depth:</span>
                                <div class="score-bar">
                                    <div class="score-fill" style="width: {questEvaluation.depth_score * 10}%"></div>
                                    <span>{questEvaluation.depth_score}/10</span>
                                </div>
                            </div>
                            <div class="score-item">
                                <span>Authenticity:</span>
                                <div class="score-bar">
                                    <div class="score-fill" style="width: {questEvaluation.authenticity_score * 10}%"></div>
                                    <span>{questEvaluation.authenticity_score}/10</span>
                                </div>
                            </div>
                        </div>
                    </div>
                    
                    <div class="feedback-section">
                        <h4><i class="fas fa-comment"></i> AI Feedback</h4>
                        <p class="feedback-text">{questEvaluation.feedback}</p>
                        
                        {#if questEvaluation.strengths && questEvaluation.strengths.length > 0}
                            <div class="strengths">
                                <h5><i class="fas fa-thumbs-up"></i> Strengths</h5>
                                <ul>
                                    {#each questEvaluation.strengths as strength}
                                        <li>{strength}</li>
                                    {/each}
                                </ul>
                            </div>
                        {/if}
                        
                        {#if questEvaluation.areas_for_growth && questEvaluation.areas_for_growth.length > 0}
                            <div class="growth-areas">
                                <h5><i class="fas fa-seedling"></i> Areas for Growth</h5>
                                <ul>
                                    {#each questEvaluation.areas_for_growth as area}
                                        <li>{area}</li>
                                    {/each}
                                </ul>
                            </div>
                        {/if}
                    </div>
                    
                    <div class="modal-actions">
                        <button class="continue-btn" on:click={closeEvaluationModal}>
                            <i class="fas fa-arrow-right"></i>
                            Continue Journey
                        </button>
                    </div>
                </div>
            </div>
        </div>
    {/if}
</div>

<style>
    :global(body) {
        margin: 0;
        font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif;
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
        gap: 2rem;
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

    .quest-indicator {
        color: #667eea !important;
        font-weight: 600;
        margin-top: 0.25rem;
    }

    .quest-indicator i {
        animation: pulse 2s infinite;
    }

    @keyframes pulse {
        0% { opacity: 1; }
        50% { opacity: 0.6; }
        100% { opacity: 1; }
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
        background: #4caf50;
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

    .quest-banner {
        background: linear-gradient(135deg, #667eea, #764ba2);
        color: white;
        padding: 1rem 1.5rem;
        border-bottom: 1px solid rgba(255, 255, 255, 0.2);
    }

    .quest-banner-content {
        display: flex;
        justify-content: space-between;
        align-items: center;
    }

    .quest-banner h4 {
        margin: 0 0 0.5rem 0;
        display: flex;
        align-items: center;
        gap: 0.5rem;
    }

    .quest-banner p {
        margin: 0;
        font-style: italic;
        opacity: 0.9;
    }

    .cancel-quest-btn {
        background: rgba(255, 255, 255, 0.2);
        color: white;
        border: none;
        padding: 0.5rem 1rem;
        border-radius: 15px;
        font-size: 0.9rem;
        cursor: pointer;
        transition: all 0.3s ease;
    }

    .cancel-quest-btn:hover {
        background: rgba(255, 255, 255, 0.3);
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
        min-height: 500px;
        overflow-y: auto;
        cursor: text;
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

    .submit-btn.quest-mode {
        background: linear-gradient(135deg, #4caf50, #45a049);
    }

    .submit-btn:hover:not(:disabled) {
        transform: translateY(-2px);
        box-shadow: 0 8px 20px rgba(102, 126, 234, 0.3);
    }

    .submit-btn.quest-mode:hover:not(:disabled) {
        box-shadow: 0 8px 20px rgba(76, 175, 80, 0.3);
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

    .entry-card.quest-entry {
        border-left: 4px solid #667eea;
        background: rgba(102, 126, 234, 0.05);
    }

    .entry-header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin-bottom: 0.5rem;
    }

    .entry-date {
        color: #667eea;
        font-size: 0.8rem;
        font-weight: 600;
    }

    .entry-type {
        color: #667eea;
        font-size: 0.7rem;
        background: rgba(102, 126, 234, 0.1);
        padding: 0.2rem 0.5rem;
        border-radius: 10px;
        display: flex;
        align-items: center;
        gap: 0.3rem;
    }

    .quest-context {
        color: #555;
        font-size: 0.8rem;
        font-style: italic;
        margin-bottom: 0.5rem;
        padding: 0.5rem;
        background: rgba(102, 126, 234, 0.1);
        border-radius: 8px;
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
        display: flex;
        gap: 1rem;
    }

    .quests-overlay, .modal-overlay {
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

    .quests-panel, .evaluation-modal {
        background: white;
        border-radius: 20px;
        max-width: 700px;
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

    .quests-header, .modal-header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        padding: 1.5rem;
        border-bottom: 1px solid #eee;
    }

    .quests-header h3, .modal-header h3 {
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

    .quest-tabs {
        display: flex;
        border-bottom: 1px solid #eee;
    }

    .tab-btn {
        flex: 1;
        background: none;
        border: none;
        padding: 1rem;
        cursor: pointer;
        font-weight: 500;
        color: #666;
        transition: all 0.3s ease;
        border-bottom: 2px solid transparent;
    }

    .tab-btn.active {
        color: #667eea;
        border-bottom-color: #667eea;
        background: rgba(102, 126, 234, 0.05);
    }

    .tab-btn:hover:not(.active) {
        background: rgba(0, 0, 0, 0.05);
    }

    .quests-content, .modal-content {
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
        font-weight: 500;
    }

    .quest-actions, .quest-completion {
        display: flex;
        justify-content: space-between;
        align-items: center;
    }

    .quest-reward {
        color: #667eea;
        font-weight: 600;
        display: flex;
        align-items: center;
        gap: 0.3rem;
    }

    .start-quest-btn {
        background: #667eea;
        color: white;
        border: none;
        padding: 0.6rem 1.2rem;
        border-radius: 20px;
        font-weight: 600;
        cursor: pointer;
        transition: all 0.3s ease;
        display: flex;
        align-items: center;
        gap: 0.5rem;
    }

    .start-quest-btn:hover {
        background: #5a6fd8;
        transform: translateY(-1px);
    }

    .completion-info {
        display: flex;
        flex-direction: column;
        gap: 0.5rem;
    }

    .completed-badge {
        color: #4caf50;
        font-weight: 600;
        display: flex;
        align-items: center;
        gap: 0.3rem;
    }

    .xp-earned {
        color: #ff9800;
        font-weight: 600;
        display: flex;
        align-items: center;
        gap: 0.3rem;
        font-size: 0.9rem;
    }

    .empty-state {
        text-align: center;
        padding: 2rem;
        color: #666;
    }

    .empty-state i {
        font-size: 3rem;
        margin-bottom: 1rem;
        color: #ccc;
    }

    /* Evaluation Modal Styles */
    .score-summary {
        background: rgba(102, 126, 234, 0.05);
        border-radius: 12px;
        padding: 1.5rem;
        margin-bottom: 1.5rem;
    }

    .xp-awarded {
        text-align: center;
        margin-bottom: 1.5rem;
    }

    .xp-awarded i {
        font-size: 2rem;
        color: #ff9800;
        margin-right: 0.5rem;
    }

    .xp-awarded span {
        font-size: 1.5rem;
        font-weight: 700;
        color: #333;
    }

    .score-breakdown {
        display: flex;
        flex-direction: column;
        gap: 1rem;
    }

    .score-item {
        display: flex;
        align-items: center;
        gap: 1rem;
    }

    .score-item span:first-child {
        min-width: 100px;
        font-weight: 600;
        color: #555;
    }

    .score-bar {
        flex: 1;
        height: 20px;
        background: rgba(0, 0, 0, 0.1);
        border-radius: 10px;
        position: relative;
        display: flex;
        align-items: center;
        justify-content: flex-end;
        padding-right: 0.5rem;
    }

    .score-fill {
        position: absolute;
        left: 0;
        top: 0;
        height: 100%;
        background: linear-gradient(90deg, #4caf50, #45a049);
        border-radius: 10px;
        transition: width 0.5s ease;
    }

    .score-bar span {
        font-size: 0.8rem;
        font-weight: 600;
        color: #333;
        z-index: 1;
    }

    .feedback-section {
        margin-bottom: 1.5rem;
    }

    .feedback-section h4 {
        color: #333;
        margin-bottom: 1rem;
        display: flex;
        align-items: center;
        gap: 0.5rem;
    }

    .feedback-text {
        color: #555;
        line-height: 1.6;
        margin-bottom: 1.5rem;
        background: rgba(0, 0, 0, 0.05);
        padding: 1rem;
        border-radius: 8px;
    }

    .strengths, .growth-areas {
        margin-bottom: 1rem;
    }

    .strengths h5, .growth-areas h5 {
        color: #333;
        margin-bottom: 0.5rem;
        display: flex;
        align-items: center;
        gap: 0.5rem;
        font-size: 1rem;
    }

    .strengths h5 i {
        color: #4caf50;
    }

    .growth-areas h5 i {
        color: #ff9800;
    }

    .strengths ul, .growth-areas ul {
        margin: 0;
        padding-left: 1.5rem;
        color: #555;
    }

    .strengths li, .growth-areas li {
        margin-bottom: 0.3rem;
    }

    .modal-actions {
        text-align: center;
        padding-top: 1rem;
        border-top: 1px solid #eee;
    }

    .continue-btn {
        background: linear-gradient(135deg, #667eea, #764ba2);
        color: white;
        border: none;
        padding: 1rem 2rem;
        border-radius: 25px;
        font-weight: 600;
        cursor: pointer;
        transition: all 0.3s ease;
        display: inline-flex;
        align-items: center;
        gap: 0.5rem;
        font-size: 1rem;
    }

    .continue-btn:hover {
        transform: translateY(-2px);
        box-shadow: 0 8px 20px rgba(102, 126, 234, 0.3);
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

        .avatar-container {
            width: 80px;
            height: 80px;
        }

        .rive-canvas, .avatar {
            width: 80px;
            height: 80px;
        }

        .avatar {
            font-size: 2rem;
        }

        .quest-banner-content {
            flex-direction: column;
            gap: 1rem;
            align-items: flex-start;
        }

        .quest-tabs {
            flex-direction: column;
        }

        .score-item {
            flex-direction: column;
            align-items: flex-start;
            gap: 0.5rem;
        }

        .score-item span:first-child {
            min-width: auto;
        }
    }
</style>