# Application Packages: pi-web-ui

> **Note (v0.71.0, April 2026):** The `pi-mom` (Slack bot) and `pi-pods` (GPU management CLI) packages were removed from the pi-mono monorepo. Only `pi-web-ui` remains as an application package.

This document covers the application-level package in the pi-mono monorepo: the browser-based chat interface.

---

## pi-web-ui: Browser Chat Interface

Package: `@mariozechner/pi-web-ui` (v0.72.1)
License: MIT
Source: `packages/web-ui/`

pi-web-ui provides reusable web UI components for building AI chat interfaces in the browser. It is built with [mini-lit](https://github.com/badlogic/mini-lit) web components and Tailwind CSS v4.

### Dependencies

- `@mariozechner/pi-ai` -- LLM provider abstraction
- `@mariozechner/pi-tui` -- Reused for fuzzy matching utilities
- `@mariozechner/pi-agent-core` -- Agent state machine (peer dependency)
- `@mariozechner/mini-lit` -- Lightweight Lit-based component framework (peer dependency)
- `lit` -- Web components library (peer dependency)
- `pdfjs-dist`, `docx-preview`, `jszip`, `xlsx` -- Document parsing
- `ollama`, `@lmstudio/sdk` -- Local LLM provider integration
- `lucide` -- Icon library

### Architecture

```
ChatPanel
├── AgentInterface (messages, input, model selector)
└── ArtifactsPanel (HTML, SVG, Markdown artifacts)
         │
         ▼
    Agent (from pi-agent-core)
    - State management
    - Event emission
    - Tool execution
         │
         ▼
    AppStorage
    ├── SettingsStore (key-value settings)
    ├── ProviderKeysStore (API keys per provider)
    ├── SessionsStore (chat sessions with metadata)
    └── CustomProvidersStore (Ollama, LM Studio, vLLM configs)
              │
       IndexedDBStorageBackend
```

### ChatPanel (`src/ChatPanel.ts`)

The top-level component, registered as `<pi-chat-panel>`. It orchestrates:

1. **AgentInterface** -- The chat message list, input area, model selector, thinking level selector, and attachment handling
2. **ArtifactsPanel** -- Side panel (or overlay on mobile) for interactive artifacts

Key behavior:
- Responsive layout: at 800px breakpoint, switches between side-by-side (desktop) and overlay (mobile) modes for the artifacts panel
- Artifacts panel auto-opens when a new artifact is created
- A floating badge shows the artifact count when the panel is collapsed
- `setAgent(agent, config)` wires up the entire system: creates the AgentInterface, ArtifactsPanel, registers tool renderers, and sets up runtime providers

Configuration callbacks:
- `onApiKeyRequired(provider)` -- Prompt for API key when needed
- `onBeforeSend()` -- Hook before sending messages
- `onCostClick()` -- Handle cost display click
- `sandboxUrlProvider()` -- Custom sandbox URL (for browser extensions like sitegeist)
- `toolsFactory(agent, agentInterface, artifactsPanel, runtimeProvidersFactory)` -- Add custom tools

### Components

The package exports a large number of web components:

- **AgentInterface** -- Lower-level chat interface for custom layouts. Properties: `session` (Agent), `enableAttachments`, `enableModelSelector`, `enableThinkingSelector`, `showThemeToggle`
- **MessageList** / **MessageEditor** -- Message display and editing
- **StreamingMessageContainer** -- Container for streaming assistant responses
- **AttachmentTile** -- Preview tile for attached files
- **Input** -- Text input component
- **ConsoleBlock** -- Code/console output display
- **ThinkingBlock** -- Expandable thinking content display
- **ExpandableSection** -- Collapsible section wrapper
- **CustomProviderCard** / **ProviderKeyInput** -- Custom provider configuration UI

### Message Types

Beyond standard LLM messages (user, assistant, toolResult), pi-web-ui defines:

- `UserMessageWithAttachments` -- role `"user-with-attachments"` with an `attachments` array. Type guard: `isUserMessageWithAttachments()`
- `ArtifactMessage` -- role `"artifact"` with `action` (create/update/delete), `filename`, and `content`. Used for session persistence of artifacts. Type guard: `isArtifactMessage()`

Custom message types can be added via TypeScript declaration merging on `CustomAgentMessages` from `@mariozechner/pi-agent-core`.

### Message Transformation

`defaultConvertToLlm(messages)` transforms application messages to LLM-compatible format:
- `UserMessageWithAttachments` -- Converts to user message with image and text content blocks via `convertAttachments()`
- `ArtifactMessage` -- Filtered out (UI-only, not sent to LLM)
- Standard messages -- Passed through

### Tools

**JavaScript REPL** (`createJavaScriptReplTool()`): Runs JavaScript in a sandboxed browser iframe. Configured with runtime providers for artifact and attachment access.

**Extract Document** (`createExtractDocumentTool()`): Extracts text from documents at URLs. Supports CORS proxy configuration.

**Artifacts**: Built into ArtifactsPanel. Supports HTML, SVG, Markdown, text, JSON, images, PDF, DOCX, XLSX. Each artifact type has a dedicated renderer (HtmlArtifact, SvgArtifact, MarkdownArtifact, TextArtifact, ImageArtifact).

> **Fix (v0.69.0):** SVG artifact previews are now rendered inside sandboxed iframes,
> preventing SVG `<script>` payloads from escaping the sandbox boundary.

Custom tool renderers can be registered via `registerToolRenderer(name, renderer)`.

### Sandbox Runtime Providers

The sandbox system runs code in isolated iframes with controlled access:

- `ArtifactsRuntimeProvider` -- Provides read or read-write access to artifacts
- `AttachmentsRuntimeProvider` -- Provides access to user-attached files
- `ConsoleRuntimeProvider` -- Captures console output from sandboxed code
- `FileDownloadRuntimeProvider` -- Handles file downloads from sandbox
- `RuntimeMessageBridge` / `RUNTIME_MESSAGE_ROUTER` -- Communication layer between main page and sandbox iframe

### Storage

All persistence uses IndexedDB via `IndexedDBStorageBackend`:

- **SettingsStore** -- Key-value settings (proxy configuration, etc.)
- **ProviderKeysStore** -- API keys indexed by provider name
- **SessionsStore** -- Chat sessions with separate metadata store. Sessions are saved with full message history and loaded via metadata listing (sorted by lastModified)
- **CustomProvidersStore** -- Custom LLM provider configurations (Ollama, LM Studio, vLLM, OpenAI-compatible). Each provider has an `id`, `name`, `type`, and `baseUrl`

Global storage is managed via `setAppStorage(storage)` / `getAppStorage()`.

### CORS Proxy

Browser environments face CORS restrictions when calling LLM APIs directly. The proxy system:
- `createStreamFn()` creates a stream function that reads proxy settings on each call
- `shouldUseProxyForProvider(provider)` determines which providers need proxying (zai always, anthropic only for OAuth tokens)
- `isCorsError()` detects CORS errors for user-friendly error messages

### Dialogs

- **SettingsDialog** -- Tabbed settings with ProvidersModelsTab, ProxyTab, ApiKeysTab
- **SessionListDialog** -- Browse and manage chat sessions
- **ApiKeyPromptDialog** -- Modal prompt for API key entry
- **ModelSelector** -- Fuzzy-searchable model selection dialog

### Internationalization

Simple string-based i18n via `i18n(key)`, `setLanguage(lang)`, and a `translations` object. Custom translations are added as key-value mappings.

---

## pi-mom (Removed)

> **Removed in v0.71.0 (April 2026).** The `pi-mom` Slack bot package was removed from the pi-mono monorepo. It is no longer published or maintained as part of this project.

---

## pi-pods (Removed)

> **Removed in v0.71.0 (April 2026).** The `pi-pods` GPU pod management CLI package was removed from the pi-mono monorepo. It is no longer published or maintained as part of this project.
