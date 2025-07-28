# 🧠 Devlogue — A Journaling RPG

**Devlogue** is a journaling RPG web app that gamifies self-reflection and knowledge exploration. As you write about your day, debug your thoughts, or explore ideas, a game world grows around you — with quests, NPCs (not implemented), a world map(not implemented), and your own evolving avatar(not implemented).

> ✨ Built for journaling enthusiasts, and RPG lovers — Devlogue turns your thoughts into adventures.

![Animated demo of the Devlogue web app showing a user journaling in a rich text editor, with a sidebar displaying AI-generated prompts and quests. The interface features a clean, modern design with playful RPG-inspired elements. The overall tone is inviting and encourages exploration and self-reflection.](https://github.com/AswinBehera/Devlogue/blob/main/static/devlogue.gif)

---

## 🧩 Key Features

- 📝 **Rich Text Journaling** using Quill.js with autosave.
- 🤖 **AI-powered Journal Companion** (Gemini AI) to generate:
  - Reflective prompts
  - Summary of past entries
  - Quests derived from writing patterns
- 🧍‍♂️ **Animated Rive Avatar** that reacts to entries (not implemented) and clicks.

---

## 🚀 Installation Guide

You’ll need:

- Node.js v18+
- Git (optional but helpful)
- A modern browser (Chrome, Firefox, Edge)

### 🖥️ For all platforms (Linux, macOS, Windows):

# Devlogue

A journaling app with AI-powered reflection quests and themes.

## Getting Started

### 1. Clone the Repository

```sh
git clone https://github.com/yourusername/devlogue.git
cd devlogue
```

### 2. Install Dependencies

```sh
npm install
```

### 3. Set Up Ollama for AI Integration

This app uses [Ollama](https://ollama.com/) to run local LLMs for journal analysis.

#### a. Install Ollama

Follow the instructions at [ollama.com/download](https://ollama.com/download) to install Ollama for your platform.

#### b. Pull a Model

You can use `deepseek-r1` (recommended) or any other supported model.  
To pull `deepseek-r1`, run:

```sh
ollama pull deepseek-r1
```

Or, for another model (e.g., `llama3`):

```sh
ollama pull llama3
```

#### c. Start Ollama

Ollama usually runs automatically after installation. If not, start it with:

```sh
ollama serve
```

#### d. Configure the Model in Code

By default, the app uses `deepseek-r1`.  
If you want to use a different model, open  
`src/routes/+page.svelte`  
and find this section in the `analyzeEntry` function:

```js
body: JSON.stringify({
    model: "deepseek-r1", // or another model you have pulled
    prompt,
    stream: false,
}),
```

Change `"deepseek-r1"` to your preferred model name (e.g., `"llama3"`).

---

### 4. Run the App

```sh
npm run dev
```

Visit [http://localhost:5173](http://localhost:5173) in your browser.

---

## Notes

- Make sure Ollama is running and the model you specify is pulled.
- The AI features will not work if Ollama is not running or the model is not available.
- For best results, use a model that supports instruction-following and JSON output.

---

## License

Apache 2.0
