# AI Search — Browser-Based LLM Agent

A fully client-side, single-file agentic AI interface that accepts natural-language prompts, breaks them down into executable JavaScript, runs the code in the browser, validates the result with a second LLM pass, and automatically retries with corrected code if the result is unsatisfactory. No backend required.

---

## Table of Contents

- [Overview](#overview)
- [How It Works](#how-it-works)
- [Agent Loop Architecture](#agent-loop-architecture)
- [Features](#features)
- [Repository Structure](#repository-structure)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
  - [API Keys](#api-keys)
  - [System Prompt](#system-prompt)
  - [Adding More API Integrations](#adding-more-api-integrations)
- [UI Walkthrough](#ui-walkthrough)
- [Dependencies](#dependencies)
- [Security Notes](#security-notes)

---

## Overview

This project is a proof-of-concept for a **browser-native reasoning agent**. Given a natural-language question, the agent:

1. Analyses the task and writes JavaScript to solve it — for example, calling the Google Custom Search API.
2. Executes that code securely inside the browser using dynamic ES module imports.
3. Sends the result to a **validator LLM** that checks whether the question was fully answered.
4. If the validator says the result is incomplete or wrong, it explains the error and the agent rewrites and reruns the code — up to a configurable number of attempts.

Everything runs in a single `index.html` file with zero server-side code.

---

## How It Works

### Splash Screen

On load, a splash screen presents the agent's capabilities with a typewriter animation. Clicking the lightbulb icon starts the animation; clicking **Continue** enters the main interface.

### Agent & Validator Prompts

Two distinct LLM system prompts power the loop:

**Agent prompt (`agentPrompt`)** — instructs the model to act as a JavaScript developer. Given API details from the selected tools and the user's question, it must produce a single, self-contained `export async function run(params)` code block that fetches data and returns a structured result.

**Validator prompt (`validatorPrompt`)** — receives only the original user question, the last code output, and the last agent message. It replies with either:
- `🟢 DONE` — the question is fully answered; the response is rendered to the user.
- `🔴 REVISE` — the result is incomplete or erroneous; it explains what went wrong so the agent can correct the code on the next attempt.

### Code Execution

The agent's JavaScript is extracted from the response (anything inside a ` ```js ``` ` fence), wrapped in a `Blob`, and imported as a live ES module. This gives the code full access to `fetch` (intercepted by a logging wrapper) while keeping execution inside the browser sandbox.

---

## Agent Loop Architecture

```
User submits question
        │
        ▼
┌───────────────────────────────┐
│  Agent LLM (gpt-4o-mini)      │
│  System: agentPrompt          │
│  + full message history       │
│                               │
│  Output: analysis + JS code   │
└──────────────┬────────────────┘
               │
               │  Contains 🟢? → render answer, stop
               │
               ▼
    Extract ```js block
               │
               ▼
  Wrap in Blob → import() as ES module
               │
               ▼
      await module.run(params)
               │
        ┌──────┴───────┐
      success        error
        │               │
        ▼               ▼
  result JSON      error stack
        │               │
        └──────┬─────────┘
               ▼
┌───────────────────────────────┐
│  Validator LLM (gpt-4o-mini)  │
│  System: validatorPrompt      │
│  + original question + result │
│                               │
│  🟢 DONE → show answer, stop  │
│  🔴 REVISE → feed back error  │
└──────────────┬────────────────┘
               │
               ▼
     Next attempt (max 3)
```

---

## Features

- **Zero backend** — everything runs in the browser; no server, no build step.
- **Agentic retry loop** — automatically rewrites and reruns code when the validator flags an incomplete or incorrect result (configurable attempt limit, default 3).
- **Google Custom Search integration** — toggle on/off with a single button; enter your API key and Search Engine ID in the sidebar.
- **Customisable system prompt** — the system prompt textarea is fully editable before submitting, letting you add extra API documentation or constraints on the fly.
- **Streamed LLM responses** — uses `asyncLLM` for real-time token streaming so you see the agent's reasoning as it types.
- **Syntax-highlighted code blocks** — code in both agent responses and results is rendered with `highlight.js` using the GitHub Dark theme.
- **Collapsible step cards** — each message in the conversation (user, developer, result, error, validator) is shown as a colour-coded, collapsible Bootstrap card.
- **Form state persistence** — API keys and prompt values are saved locally via `saveform` so they survive page reloads.
- **OAuth support** — the sidebar's API panel architecture supports Google OAuth flows for APIs that need token-based auth.
- **Dark mode** — integrated via `@gramex/ui` dark-theme module.
- **Sample questions** — randomly selected example prompts from the active tool set are shown as one-click buttons.

---

## Repository Structure

```
.
├── index.html      # The entire application — UI, styles, and logic
├── unnamed.png     # Background texture / wallpaper image
├── LICENSE
└── README.md
```

The application is intentionally a single file. All JavaScript is inlined as an ES module inside `<script type="module">`, and all styles are inlined in `<style>`. No bundler, no Node.js, no framework.

---

## Getting Started

### Option 1 — Serve locally

Because the app uses dynamic `import()` of `Blob` URLs, it must be served over HTTP (not opened as a `file://` URL) in most browsers.

```bash
# Python (any machine with Python 3)
python -m http.server 8080
# then open http://localhost:8080
```

```bash
# Node.js
npx serve .
```

### Option 2 — Deploy statically

Upload `index.html` and `unnamed.png` to any static host (GitHub Pages, Netlify, Vercel, an S3 bucket, etc.). No server configuration is needed.

---

## Configuration

### API Keys

#### LLM (required)

The agent calls an OpenAI-compatible chat completions endpoint. Replace the `BASE_URL`, `API_KEY`, and `MODEL` constants in the submit handler inside `index.html`:

```js
const BASE_URL = "https://aipipe.org/openai/v1";  // or any OpenAI-compatible endpoint
const API_KEY  = "YOUR_KEY_HERE";
const MODEL    = "gpt-4o-mini";                    // or any model your endpoint supports
const ATTEMPTS = 3;                                // max retry attempts
```

> **Never commit real API keys to a public repository.** Load them from user input or a secure token service before deploying publicly.

#### Google Custom Search (optional)

To enable web search:

1. Click **Enable Google Custom Search** in the main interface.
2. Enter your [Google CSE API key](https://developers.google.com/custom-search/v1/introduction) in the sidebar field labelled *Google CSE API key*.
3. Enter your [Custom Search Engine ID (cx)](https://developers.google.com/custom-search/v1/using_rest#search_engine_id).

Keys are saved to `localStorage` by `saveform` and reloaded on the next visit.

### System Prompt

The **System Prompt** textarea at the top of the form is populated automatically when you enable/disable tools. You can edit it freely before submitting — useful for adding extra context, restricting the agent's scope, or providing additional API documentation.

### Adding More API Integrations

New tools are defined in the `demos` array at the top of the script block. Each entry follows this schema:

```js
{
  icon: "bootstrap-icon-name",   // e.g. "search", "envelope", "calendar"
  title: "Tool Display Name",
  description: "Short description shown in the sidebar.",
  prompt: "API documentation injected into the agent's system prompt.",
  questions: [                   // Sample questions shown as one-click buttons
    "Example question 1",
    "Example question 2",
  ],
  params: [                      // Credentials/config collected from the user
    {
      label: "Human-readable label",
      link:  "https://docs.example.com/get-key",  // 'Get token' help link
      required: true,
      key:  "param_key_name",    // accessed in run(params) as params.param_key_name
      type: "password",          // or "text"
    },
  ],
}
```

The agent receives all selected tools' `prompt` strings concatenated into its system message, and each tool's `params` values are passed to `run(params)` at execution time.

---

## UI Walkthrough

| Element | Description |
|---|---|
| Splash screen | Typewriter introduction; click 💡 to start the animation, then **Continue** to enter the app. |
| **Enable Google Custom Search** button | Toggles Google CSE on/off. Turns red when enabled; credential fields appear in the sidebar. |
| Sidebar | Lists all non-Google tool panels with their credential input fields. |
| **System Prompt** textarea | Pre-filled from active tools; fully editable. Persists across reloads. |
| **Sample Questions** | Up to 6 randomly selected example prompts from active tools. Clicking one submits it immediately. |
| **Results** area | Conversation rendered as colour-coded, collapsible step cards. |
| Status bar | Shows the URL being fetched while generated code is running; hidden otherwise. |
| **Question** textarea | Free-text input for your own question. |
| **Submit** button | Starts the agent loop. |

### Step card colour coding

| Colour | Role | Meaning |
|---|---|---|
| Blue | `user` | Your question or follow-up |
| Green | `developer` | Agent's reasoning and generated JS code |
| Teal | `result` | JSON output returned by the executed code |
| Red | `error` | JavaScript runtime error or API failure |
| Yellow | `validator` | Validator's assessment (🟢 DONE or 🔴 REVISE) |

---

## Dependencies

All loaded from CDN — no `npm install` required.

| Library | Version | Purpose |
|---|---|---|
| Bootstrap | 5.3.3 | Layout, components, utility classes |
| Bootstrap Icons | 1.11.3 | Step card and sidebar icons |
| `lit-html` | 3 | Efficient declarative DOM rendering |
| `asyncLLM` | 2 | Streaming LLM responses via async iterators |
| `marked` | 13 | Markdown → HTML for agent and validator responses |
| `highlight.js` | 11.9.0 | Syntax highlighting for code blocks (GitHub Dark theme) |
| `saveform` | 1 | Persists form field values to `localStorage` |
| `@gramex/ui` | 0.3 | Dark mode toggle |
| Google GSI client | latest | Google OAuth token flow (loaded on demand) |

---

## Security Notes

- **API keys in source code** — the `API_KEY` constant is a placeholder. Load keys from user input, environment injection at build time, or a secure token service before any public deployment.
- **Dynamic code execution** — LLM-generated JavaScript is executed as a browser ES module. While sandboxed by the browser's same-origin model, treat generated code with the same caution as any dynamic script; a compromised or hallucinating model could produce unexpected network requests.
- **CORS** — the Google CSE endpoint and `aipipe.org` both support CORS from the browser. If you swap in a different API that doesn't allow cross-origin requests, a thin CORS proxy will be needed.
- **`fetch` interception** — `globalThis.customFetch` wraps every `fetch` call made by agent-generated code solely to display the outbound URL in the status bar. It does not modify request headers or credentials.
