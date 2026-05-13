# Application Packages

This document covers the **application-level packages** in the pi-mono monorepo at **v0.74.0** (commit `1eee081e`, tagged 2026-05-07). At this version, the monorepo ships **five** packages under the `@earendil-works/*` npm scope:

| Package | Role | Layer |
|---------|------|-------|
| [`@earendil-works/pi-ai`](packages/ai) | Unified multi-provider LLM API | Foundation |
| [`@earendil-works/pi-agent-core`](packages/agent) | Agent runtime with tool calling and state | Foundation |
| [`@earendil-works/pi-tui`](packages/tui) | Terminal UI library with differential rendering | Foundation |
| [`@earendil-works/pi-coding-agent`](packages/coding-agent) | Interactive coding agent CLI | Application |
| [`@earendil-works/pi-web-ui`](packages/web-ui) | Web components for AI chat interfaces | Application |

Two earlier packages (`pi-mom`, `pi-pods`) were **removed** from the monorepo in April 2026 and are not part of v0.74.0. They are covered at the end of this document for historical reference. For Slack/chat automation, the project now points users to the **external** [`earendil-works/pi-chat`](https://github.com/earendil-works/pi-chat) repository (see [root README](https://github.com/earendil-works/pi-mono/blob/v0.74.0/README.md)).

This document focuses on **`pi-web-ui`** — the only browser-facing application package in v0.74.0. For the CLI application (`pi-coding-agent`), see [04-pi-coding-agent.md](./04-pi-coding-agent.md).

---

## pi-web-ui: Browser Chat Interface

- **Package:** `@earendil-works/pi-web-ui` v0.74.0
- **License:** MIT
- **Source:** [`packages/web-ui/`](https://github.com/earendil-works/pi-mono/tree/v0.74.0/packages/web-ui)
- **Author:** Mario Zechner
- **Description (from `package.json`):** *"Reusable web UI components for AI chat interfaces powered by @earendil-works/pi-ai"*

pi-web-ui is a library of [mini-lit](https://github.com/badlogic/mini-lit) (Lit-based) web components and Tailwind CSS v4 styles for embedding a complete AI chat UI into any web application. It supplies high-level pieces (the `ChatPanel` orchestrator), low-level pieces (message renderers, dialogs, tool renderers), an **IndexedDB-backed persistence layer**, and a **sandboxed code execution / artifacts system** for safely running LLM-generated JavaScript and HTML in the browser.

The package ships TypeScript sources compiled with `tsc` and a pre-built Tailwind CSS bundle (`@earendil-works/pi-web-ui/app.css`).

### Dependencies

From [`package.json`](https://github.com/earendil-works/pi-mono/blob/v0.74.0/packages/web-ui/package.json):

**Runtime dependencies**
- `@earendil-works/pi-ai` `^0.74.0` — LLM provider abstraction (`getModel`, `streamSimple`, `Usage`)
- `@earendil-works/pi-tui` `^0.74.0` — Reused for shared utilities
- `@lmstudio/sdk` `^1.5.0` and `ollama` `^0.6.0` — Local LLM provider integration
- `typebox` `^1.1.24` — JSON-Schema friendly tool parameter schemas (migrated from `@sinclair/typebox` in v0.69.0)
- `pdfjs-dist` `5.4.394`, `docx-preview` `^0.3.7`, `jszip` `^3.10.1`, `xlsx` `0.20.3` — Document parsing (PDF, DOCX, ZIP, XLSX)
- `lucide` `^0.544.0` — Icon library

**Peer dependencies**
- `@mariozechner/mini-lit` `^0.2.0` — Lightweight Lit-based component framework
- `lit` `^3.3.1` — Web Components library

**Note:** The `Agent` class itself is **not** in this package. As of v0.31.0, it moved to `@earendil-works/pi-agent-core` (which pi-web-ui re-exports types from). See [03-pi-agent-core.md](./03-pi-agent-core.md).

### Public API surface

`src/index.ts` exports roughly 80 symbols across components, dialogs, storage, tools, runtime providers, and utilities. The major groupings are:

- **Top-level orchestrator:** `ChatPanel`
- **Components:** `AgentInterface`, `AttachmentTile`, `ConsoleBlock`, `CustomProviderCard`, `ExpandableSection`, `Input`, `MessageEditor`, `MessageList`, `ProviderKeyInput`, `StreamingMessageContainer`, `ThinkingBlock`, `SandboxIframe`
- **Message renderers:** `UserMessage`, `AssistantMessage`, `ToolMessage`, `ToolMessageDebugView`, `AbortedMessage`, plus the registry: `registerMessageRenderer`, `getMessageRenderer`, `renderMessage`
- **Sandbox runtime providers:** `ArtifactsRuntimeProvider`, `AttachmentsRuntimeProvider`, `ConsoleRuntimeProvider`, `FileDownloadRuntimeProvider`, `RuntimeMessageBridge`, `RUNTIME_MESSAGE_ROUTER`
- **Dialogs:** `ModelSelector`, `SettingsDialog` (+ `SettingsTab`, `ProxyTab`, `ApiKeysTab`), `ProvidersModelsTab`, `SessionListDialog`, `ApiKeyPromptDialog`, `CustomProviderDialog`, `AttachmentOverlay`, `PersistentStorageDialog`
- **Storage:** `AppStorage`, `getAppStorage`, `setAppStorage`, `IndexedDBStorageBackend`, `Store`, plus four stores: `SettingsStore`, `ProviderKeysStore`, `SessionsStore`, `CustomProvidersStore`
- **Artifacts:** `ArtifactsPanel`, `ArtifactElement`, `ArtifactPill`, `ArtifactsToolRenderer`, `HtmlArtifact`, `SvgArtifact`, `MarkdownArtifact`, `TextArtifact`, `ImageArtifact`
- **Tools:** `createJavaScriptReplTool`, `javascriptReplTool`, `createExtractDocumentTool`, `extractDocumentTool`, plus the renderer registry (`registerToolRenderer`, `getToolRenderer`, `renderTool`) and default renderers (`DefaultRenderer`, `BashRenderer`, `CalculateRenderer`, `GetCurrentTimeRenderer`)
- **Utilities:** `loadAttachment` (with `Attachment` type), `convertAttachments`, `defaultConvertToLlm`, `createStreamFn`, `applyProxyIfNeeded`, `shouldUseProxyForProvider`, `isCorsError`, `formatCost`/`formatTokenCount`/`formatUsage`, `i18n`/`setLanguage`/`translations`, `getAuthToken`/`clearAuthToken`

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         ChatPanel                               │
│  ┌──────────────────────────┐  ┌──────────────────────────────┐ │
│  │       AgentInterface     │  │       ArtifactsPanel         │ │
│  │  - MessageList           │  │  - HtmlArtifact (sandboxed)  │ │
│  │  - StreamingContainer    │  │  - SvgArtifact, ImageArtifact│ │
│  │  - MessageEditor + tiles │  │  - MarkdownArtifact          │ │
│  │  - Model/Thinking pickers│  │  - PdfArtifact, DocxArtifact │ │
│  │  - Cost display          │  │  - ExcelArtifact, Text/Generic│ │
│  └──────────────────────────┘  └──────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                Agent (from @earendil-works/pi-agent-core)       │
│   - State management (messages, model, tools, thinkingLevel)    │
│   - Event emission (agent_start/end, turn_start/end,            │
│     message_start/update/end, state_change)                     │
│   - Tool execution loop                                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                          AppStorage                             │
│  SettingsStore │ ProviderKeysStore │ SessionsStore │            │
│                                       (+ metadata) │            │
│  CustomProvidersStore                                           │
│                              │                                  │
│                  IndexedDBStorageBackend                        │
└─────────────────────────────────────────────────────────────────┘
```

A separate, parallel subsystem — the **Sandbox & Artifacts System** — runs alongside the Agent. It is described in its own section below.

### ChatPanel

Source: [`src/ChatPanel.ts`](https://github.com/earendil-works/pi-mono/blob/v0.74.0/packages/web-ui/src/ChatPanel.ts) (~210 lines). Custom element tag: `<pi-chat-panel>`.

`ChatPanel` is the top-level orchestrator. It owns:

- An `AgentInterface` (the chat column)
- An `ArtifactsPanel` (the side / overlay panel)
- A **runtime providers factory** that produces sandbox providers wired to the live agent state

#### Responsive layout

`ChatPanel` watches `window.resize` and switches at a **`BREAKPOINT = 800px`**:

- **≥ 800px (desktop):** Chat and artifacts panel render side-by-side, each at `width: 50%` when the artifacts panel is open
- **< 800px (mobile):** The artifacts panel renders as an absolute-positioned full-screen overlay (`overlay = true`)

When at least one artifact exists and the panel is collapsed, a floating "Artifacts" badge with a count is rendered at top-center; clicking it reopens the panel. The panel auto-opens whenever a new artifact is created during the session (but not on session reload — `reconstructFromMessages` runs with the change callback temporarily detached).

#### `setAgent(agent, config?)`

The setup entry point. Wires:

```ts
await chatPanel.setAgent(agent, {
  onApiKeyRequired?: (provider) => Promise<boolean>,
  onBeforeSend?: () => void | Promise<void>,
  onCostClick?: () => void,
  onModelSelect?: () => void,
  sandboxUrlProvider?: () => string,
  toolsFactory?: (agent, agentInterface, artifactsPanel, runtimeProvidersFactory) => AgentTool<any>[],
});
```

Internally it:

1. Creates an `<agent-interface>` element, sets `session = agent`, enables attachments + model selector + thinking selector by default, and attaches the four hook callbacks
2. Creates an `ArtifactsPanel` and wires `onArtifactsChange` / `onClose` / `onOpen`
3. Registers `ArtifactsToolRenderer` against the tool name `"artifacts"` via `registerToolRenderer`
4. Constructs a **`runtimeProvidersFactory`** closure that scans `agent.state.messages` for `UserMessageWithAttachments` and returns `[AttachmentsRuntimeProvider?, ArtifactsRuntimeProvider(rw=true)]` — passed to `toolsFactory` so consumers can wire their own REPL tool
5. Sets `agent.state.tools = [artifactsPanel.tool, ...toolsFactory(...)]`
6. Calls `artifactsPanel.reconstructFromMessages(...)` to rebuild artifact state from prior `ArtifactMessage` entries on load

The `sandboxUrlProvider` callback exists for **browser extensions** with strict CSP that cannot use iframe `srcdoc`. It returns a URL like `chrome.runtime.getURL('sandbox.html')` that the iframe loads instead. The Sitegeist browser extension is the documented consumer.

### AgentInterface

Source: [`src/components/AgentInterface.ts`](https://github.com/earendil-works/pi-mono/blob/v0.74.0/packages/web-ui/src/components/AgentInterface.ts) (~400 lines). Custom element tag: `<agent-interface>`.

The lower-level chat view that subscribes to `Agent` events and renders the message column. Properties:

- `session: Agent` — The agent being viewed
- `enableAttachments`, `enableModelSelector`, `enableThinkingSelector` — Toggle UI affordances (default `true`)
- `showThemeToggle` — Default `false`
- `onApiKeyRequired`, `onBeforeSend`, `onBeforeToolCall`, `onCostClick`, `onModelSelect` — Callbacks

Behaviors:

- Uses a `ResizeObserver` on the content container to **auto-scroll to bottom** during streaming, unless the user has scrolled up (`_autoScroll` flag)
- Sets sensible defaults on the agent if not already wired: `streamFn` via `createStreamFn` (reads proxy settings from storage on each call) and `getApiKey` via `providerKeys` storage (added in v0.31.0)
- Handles all v0.31.0+ event types from `pi-agent-core`: `message_start`, `message_update`, `message_end`, `turn_start`, `turn_end`, `agent_start`, `agent_end`
- Clears the streaming container on `message_end` to prevent duplicate tool rendering (fix in v0.58.4)

### Message types and renderer registry

pi-web-ui defines two **custom message types** layered on top of `pi-agent-core`'s `AgentMessage` union:

```ts
// role: "user-with-attachments"
interface UserMessageWithAttachments {
  role: "user-with-attachments";
  content: string;
  attachments: Attachment[];
  timestamp: number;
}

// role: "artifact"
interface ArtifactMessage {
  role: "artifact";
  action: "create" | "update" | "delete";
  filename: string;
  content: string;
  timestamp: string;
}
```

Type guards `isUserMessageWithAttachments(msg)` and `isArtifactMessage(msg)` are exported.

Custom message types can be added via **TypeScript declaration merging** on the `CustomAgentMessages` interface in `@earendil-works/pi-agent-core` (the API changed in v0.31.0 — formerly `CustomMessages` in pi-web-ui itself).

The **message renderer registry** (`registerMessageRenderer`, `getMessageRenderer`, `renderMessage`) lets apps supply a `MessageRenderer` for any `MessageRole`. Standard renderers are: `UserMessage`, `AssistantMessage` (handles thinking blocks + markdown + tool messages via the *tool* renderer registry), `ToolMessage`, `AbortedMessage`.

### Message transformation: `defaultConvertToLlm`

`Agent` accepts a `convertToLlm(messages: AgentMessage[]): Message[]` function that strips UI-only metadata and translates custom message types into LLM-compatible content. pi-web-ui's default handles the two custom roles:

- `UserMessageWithAttachments` → user message whose `content` is an array of image content blocks (for image attachments) + extracted text blocks (for documents), produced by `convertAttachments(attachments)`
- `ArtifactMessage` → **filtered out** (UI-only, never sent to the LLM)
- Standard `user` / `assistant` / `toolResult` messages → passed through

Apps extend this for their own message types by composing on top of `defaultConvertToLlm` (see the example app's `customConvertToLlm`).

### Tools

pi-web-ui ships three built-in tools:

#### 1. JavaScript REPL (`createJavaScriptReplTool()`)

Source: [`src/tools/javascript-repl.ts`](https://github.com/earendil-works/pi-mono/blob/v0.74.0/packages/web-ui/src/tools/javascript-repl.ts).

Executes JavaScript in a `SandboxIframe`. Schema (typebox):

```ts
{
  title: string;  // Active-form description, e.g. "Calculating sum"
  code: string;   // JavaScript source
}
```

The tool accepts a `runtimeProvidersFactory: () => SandboxRuntimeProvider[]` which is invoked per execution to assemble the runtime environment (typically attachments + read-write artifacts). The tool's description is dynamically built from `JAVASCRIPT_REPL_TOOL_DESCRIPTION(runtimeProviderDescriptions)` so the LLM sees an accurate list of available globals. The environment supports ES2023+, all browser APIs, and `await import('https://esm.run/<pkg>')` for arbitrary npm packages (the prompt suggests XLSX, Papa Parse, Chart.js, three.js).

Output flows back as console captures + return value + a structured `files` array (for `returnDownloadableFile` calls).

#### 2. Extract Document (`createExtractDocumentTool()`)

Fetches a URL and extracts text content (PDF, DOCX, HTML). Supports a `corsProxyUrl` for browser-side fetches that hit CORS.

#### 3. Artifacts (`ArtifactsPanel.tool`)

Source: [`src/tools/artifacts/artifacts.ts`](https://github.com/earendil-works/pi-mono/blob/v0.74.0/packages/web-ui/src/tools/artifacts/artifacts.ts) (~710 lines). Element: `<artifacts-panel>`.

The artifacts tool itself is owned by the `ArtifactsPanel` instance. Schema:

```ts
{
  command: "create" | "update" | "rewrite" | "get" | "delete" | "logs";
  filename: string;       // e.g. "chart.html"
  content?: string;       // For create / rewrite
  old_str?: string;       // For update (string replacement)
  new_str?: string;       // For update
}
```

`ArtifactsPanel` instantiates a typed element per filename based on extension:

| Extension | Element |
|-----------|---------|
| `.html` | `HtmlArtifact` (sandboxed iframe; runtime providers injected) |
| `.svg` | `SvgArtifact` (blob-backed `<img>` — see v0.69.0 fix below) |
| `.md`, `.markdown` | `MarkdownArtifact` |
| `.png`/`.jpg`/`.jpeg`/`.gif`/`.webp`/`.bmp`/`.ico` | `ImageArtifact` |
| `.pdf` | `PdfArtifact` (via `pdfjs-dist`) |
| `.xlsx`, `.xls` | `ExcelArtifact` (via `xlsx`) |
| `.docx` | `DocxArtifact` (via `docx-preview`) |
| `.txt`, `.json`, `.xml`, `.yaml`, `.yml`, `.csv`, `.js`, `.ts`, `.tsx`/`.jsx`, `.py`, `.java`, `.c`/`.cpp`/`.h`, `.css`/`.scss`/`.sass`/`.less`, `.sh` | `TextArtifact` |
| anything else | `GenericArtifact` (download fallback) |

> **Security fix (v0.69.0):** SVG artifact previews are now rendered through a **blob-backed `<img>`** instead of injecting untrusted SVG markup into the page DOM. This prevents SVG `<script>` payloads from executing in the host page context (issue #3552).

#### Custom tool renderers

`registerToolRenderer(toolName, renderer: ToolRenderer)` plugs custom UI into `<tool-message>` rendering. The render function returns `{ content: TemplateResult, isCustom: boolean }` — `isCustom: true` skips the default card wrapper. Default renderers ship for `BashRenderer`, `CalculateRenderer`, `GetCurrentTimeRenderer`, plus a `DefaultRenderer` fallback.

---

## Sandbox & Artifacts System

The sandbox subsystem is the security boundary for **any LLM-generated code or markup that executes in the browser**. It is independent of any specific tool — the JavaScript REPL tool and HTML artifacts both run through it.

### `SandboxIframe`

Source: [`src/components/SandboxedIframe.ts`](https://github.com/earendil-works/pi-mono/blob/v0.74.0/packages/web-ui/src/components/SandboxedIframe.ts) (~630 lines). Custom element tag: `<sandbox-iframe>`.

A Lit element that manages a sandboxed `<iframe>` with two distinct modes:

1. **`loadContent(sandboxId, htmlContent, providers, consumers)`** — for **HTML artifacts**. Keeps the iframe mounted and displays the artifact persistently.
2. **`execute(sandboxId, code, providers, consumers, signal)`** — for the **JavaScript REPL**. Wraps `code` in a generated HTML document, runs it, captures the result, and tears the iframe down.

Loading strategies:

- **`srcdoc` mode (default):** HTML is injected into `iframe.srcdoc`. Used in normal web pages.
- **`sandboxUrlProvider` mode:** A static `sandbox.html` is loaded via URL, and content is sent in via `postMessage` after `DOMContentLoaded`. Required in browser extensions where CSP forbids inline scripts in `srcdoc`.

A `validateHtml` step runs before injection — if validation fails, the iframe shows an error panel instead of attempting to run malformed content. Script content is escaped against premature `</script>` tag closure (`escapeScriptContent`).

The iframe document is assembled by `prepareHtmlDocument(sandboxId, content, providers, opts)` which interleaves:

- The **`ConsoleRuntimeProvider`** runtime (always injected first, captures console + provides `complete()` / `onCompleted()` lifecycle)
- Each provider's `getData()` output as `window.<key>` assignments
- Each provider's `getRuntime()` stringified function, invoked with `sandboxId`
- The **`RuntimeMessageBridge`** code: `window.sendRuntimeMessage(message)` returning a `Promise` that resolves on receipt of the matching `runtime-response` from the host (or rejects after a 30-second timeout)
- The actual user / artifact HTML

### `RuntimeMessageRouter`

Source: [`src/components/sandbox/RuntimeMessageRouter.ts`](https://github.com/earendil-works/pi-mono/blob/v0.74.0/packages/web-ui/src/components/sandbox/RuntimeMessageRouter.ts).

A **singleton** (`RUNTIME_MESSAGE_ROUTER`) that owns *one* `window.addEventListener("message", ...)` for all sandboxes and routes traffic by `sandboxId`. Each sandbox is registered with:

- A set of **providers** (bidirectional; have `handleMessage(msg, respond)`)
- A set of **consumers** (one-way receivers via `handleMessage(msg)`)

For browser-extension *user scripts* (not iframe sandboxes), the router *also* installs a `chrome.runtime.onUserScriptMessage` listener that routes the same way. This is what lets the same runtime providers serve both the web-page iframe case and the extension content-script case (e.g. Sitegeist).

When the last sandbox is unregistered, both listeners are removed.

### Runtime providers

All providers implement the `SandboxRuntimeProvider` interface (source: [`SandboxRuntimeProvider.ts`](https://github.com/earendil-works/pi-mono/blob/v0.74.0/packages/web-ui/src/components/sandbox/SandboxRuntimeProvider.ts)):

```ts
interface SandboxRuntimeProvider {
  getData(): Record<string, any>;          // Injected as window.<key>
  getRuntime(): (sandboxId: string) => void; // Stringified & injected — no closures!
  handleMessage?(msg, respond): Promise<void>;
  getDescription(): string;                  // Appended to tool prompt
  onExecutionStart?(sandboxId, signal?): void;
  onExecutionEnd?(sandboxId): void;
}
```

The critical constraint: **`getRuntime()` is `toString()`'d and injected**, so it cannot reference outer scope variables, imports, or class fields. All data has to come from `window.*` (via `getData()`) or from `window.sendRuntimeMessage(...)`.

Four providers ship in the package:

| Provider | Globals injected | Direction |
|----------|------------------|-----------|
| **`ConsoleRuntimeProvider`** | Wraps `console.log/error/warn/info`; sets up `window.complete()` and `window.onCompleted(cb)` lifecycle hooks; sends original output through to the host | Sandbox → Host |
| **`AttachmentsRuntimeProvider`** | `listAttachments()`, `readTextAttachment(id)`, `readBinaryAttachment(id)` — reads from a snapshot in `window.attachments` (no messaging required, works offline) | Snapshot only |
| **`ArtifactsRuntimeProvider`** | `listArtifacts()`, `getArtifact(filename)`, `createOrUpdateArtifact(filename, content)`, `deleteArtifact(filename)` — auto-parses/stringifies `.json` files; takes a `readWrite` flag to gate mutating ops | Bidirectional |
| **`FileDownloadRuntimeProvider`** | `returnDownloadableFile(fileName, content, mimeType?)` — accepts `string` / `Uint8Array` / `Blob` / any JSON-serializable value; in extension context routes through `sendRuntimeMessage`, otherwise triggers a direct browser download | Sandbox → Host |

The `ArtifactsRuntimeProvider`'s description text (in `prompts/prompts.ts`) is exported in two variants — **`ARTIFACTS_RUNTIME_PROVIDER_DESCRIPTION_RO`** and **`ARTIFACTS_RUNTIME_PROVIDER_DESCRIPTION_RW`** — and is dynamically concatenated into the REPL tool's description so the model knows whether it can write artifacts or only read them.

### `RuntimeMessageBridge`

Source: [`src/components/sandbox/RuntimeMessageBridge.ts`](https://github.com/earendil-works/pi-mono/blob/v0.74.0/packages/web-ui/src/components/sandbox/RuntimeMessageBridge.ts).

Generates the `window.sendRuntimeMessage = async (message) => { ... }` source code for injection into either:

- A **sandbox iframe** (uses `window.parent.postMessage` + listens for `runtime-response`)
- A **user script** in a browser extension (uses `chrome.runtime.sendMessage`)

Both implementations stamp the message with `sandboxId` and a unique `messageId`, and the iframe variant times out after 30 seconds.

### Online vs. offline modes

The runtime providers are deliberately designed so that artifacts can also work in **offline mode** — i.e. when an HTML artifact is downloaded and opened standalone outside the chat app. In offline mode there's no host to message; providers fall back to:

- `AttachmentsRuntimeProvider` — always reads from the `window.attachments` snapshot (no messaging needed)
- `ArtifactsRuntimeProvider` — reads from a `window.artifacts` snapshot (read-only; mutating ops are not available)
- `FileDownloadRuntimeProvider` — uses `URL.createObjectURL` + `<a download>` instead of messaging

This is what makes downloaded HTML artifacts genuinely portable.

---

## Storage

Source: [`src/storage/`](https://github.com/earendil-works/pi-mono/tree/v0.74.0/packages/web-ui/src/storage).

All persistence is **IndexedDB-backed** through `IndexedDBStorageBackend`. Apps wire stores like this:

```ts
const settings = new SettingsStore();
const providerKeys = new ProviderKeysStore();
const sessions = new SessionsStore();
const customProviders = new CustomProvidersStore();

const backend = new IndexedDBStorageBackend({
  dbName: "my-app",
  version: 1,
  stores: [
    settings.getConfig(),
    providerKeys.getConfig(),
    sessions.getConfig(),
    SessionsStore.getMetadataConfig(),     // Separate metadata store
    customProviders.getConfig(),
  ],
});

settings.setBackend(backend);
providerKeys.setBackend(backend);
sessions.setBackend(backend);
customProviders.setBackend(backend);

const storage = new AppStorage(settings, providerKeys, sessions, customProviders, backend);
setAppStorage(storage);   // Global singleton for components to discover
```

`AppStorage` also exposes `getQuotaInfo()` and `requestPersistence()` (for `navigator.storage.persist()`).

### Stores

| Store | Purpose |
|-------|---------|
| **`SettingsStore`** | Free-form key-value settings (`proxy.enabled`, `proxy.url`, etc.) |
| **`ProviderKeysStore`** | API keys indexed by provider name (`anthropic`, `openai`, ...). Has `set` / `get` / `list` |
| **`SessionsStore`** | Two stores — **full session data** (id, title, model, thinkingLevel, full messages array, timestamps) and a separate **metadata** store (id, title, preview, messageCount, usage with cost breakdown, modelId, thinkingLevel, timestamps). Listing uses the metadata store sorted by `lastModified` |
| **`CustomProvidersStore`** | User-configured custom LLM providers: `{ id, name, type, baseUrl }` where `type` is `"ollama" \| "lmstudio" \| "vllm" \| "openai-compatible"` |

Splitting session metadata from the full message blob means session list dialogs stay fast even when individual sessions are large.

---

## CORS proxy

Browser-side calls to LLM providers run into CORS for some providers. The proxy utilities (`src/utils/proxy-utils.ts`) handle this:

- **`createStreamFn(getProxyUrl)`** — Returns a `streamFn` for `Agent` that reads proxy settings on **each** call (so toggling the setting takes effect immediately). `AgentInterface` installs this by default if not already set.
- **`shouldUseProxyForProvider(provider, apiKey?)`** — Encodes the policy: `zai` always uses the proxy; `anthropic` only uses it for OAuth tokens (keys starting with `sk-ant-oat-`); `openai-codex` and `github-copilot` were added in v0.58.4.
- **`applyProxyIfNeeded(url, provider, proxyUrl, apiKey?)`** — Rewrites a target URL through the proxy when needed.
- **`isCorsError(err)`** — Detects CORS failures for user-friendly retry prompts.

---

## Dialogs

Each dialog is registered as a custom element and exposes a static `open(...)` method.

| Dialog | Purpose |
|--------|---------|
| **`SettingsDialog`** | Tabbed settings; takes an array of `SettingsTab` (built-in: `ProvidersModelsTab`, `ProxyTab`, `ApiKeysTab`). Added `onClose` callback in v0.58.4 |
| **`SessionListDialog`** | Browse / load / delete sessions. Reads from `SessionsStore` metadata |
| **`ApiKeyPromptDialog`** | Modal "enter your <provider> API key" — returned by `onApiKeyRequired` |
| **`ModelSelector`** | Searchable model picker. Uses **subsequence fuzzy search** (v0.58.4 replaced substring matching) and is case-insensitive (v0.52.10 fix). Supports `allowedProviders` filter |
| **`CustomProviderDialog`** | Add/edit a `CustomProvider`. Exported in v0.59.0 |
| **`ProvidersModelsTab`** | Inside `SettingsDialog`: manage custom providers + view discovered models |
| **`AttachmentOverlay`** | Modal preview of attachments |
| **`PersistentStorageDialog`** | (Documented as **broken** in the package README — currently not used by the example) |

---

## Attachments

`loadAttachment(input)` (where `input` is a `File`, URL string, or `ArrayBuffer`) produces:

```ts
interface Attachment {
  id: string;
  type: "image" | "document";
  fileName: string;
  mimeType: string;
  size: number;
  content: string;          // base64
  extractedText?: string;   // For documents
  preview?: string;         // base64 preview image
}
```

Supported formats: **PDF, DOCX, XLSX, PPTX, images, plain text**. Document text is extracted at load time (so the LLM call doesn't have to re-parse).

---

## Internationalization

Trivial key-based i18n: `i18n("Loading...")` returns the matching string from `translations[currentLang]`, falling back to the key. Apps register additional locales by mutating `translations` and calling `setLanguage(lang)`.

---

## Example application

[`packages/web-ui/example/`](https://github.com/earendil-works/pi-mono/tree/v0.74.0/packages/web-ui/example) ships a complete Vite-based reference app demonstrating:

- Full storage setup (4 stores, IndexedDB, AppStorage singleton)
- Session save/load with URL-based routing (`?session=<id>`)
- Auto-title generation from the first user message
- Custom message types via `customConvertToLlm` and `registerCustomMessageRenderers`
- Adding the JavaScript REPL tool with the runtime providers factory from `setAgent`
- `SessionListDialog`, `SettingsDialog` with `ProvidersModelsTab` + `ProxyTab`, `ApiKeyPromptDialog`

Default model in the example is `claude-sonnet-4-5-20250929` (Anthropic).

The package README also points to **[sitegeist](https://sitegeist.ai)** as a real-world consumer — a browser extension that uses `sandboxUrlProvider` to satisfy extension CSP.

---

## v0.74.0 build configuration

- `npm run build` → `tsc -p tsconfig.build.json` + `tailwindcss --minify` to produce `dist/index.js`, `dist/index.d.ts`, and `dist/app.css`
- Built with **`tsc`** (not `tsgo`) — switched back in v0.58.3 because `tsgo` broke Lit decorator-driven state updates (so reactive properties were not triggering re-renders)
- Token-counting helper script: `scripts/count-prompt-tokens.ts`
- Lint/type-check: `npm run check` (biome + tsc, plus example check)

---

## Historical / external packages

The following packages are listed in DeepWiki but are **not part of v0.74.0**. They were removed from the monorepo on **2026-04-30** in commit [`0ed0d434` "remove mom and pods packages"](https://github.com/earendil-works/pi-mono/commit/0ed0d434), between releases v0.70.6 (2026-04-28) and v0.71.0 (2026-05-01). The removal commit message reads:

> *"People should check out pi-chat (earendil-works/pi-chat on GitHub), or use an older commit for mom and fork."*

### pi-pods — GPU pod manager (removed)

A CLI tool for managing **vLLM** deployments on remote GPU servers via SSH. Per the DeepWiki summary, it provided:

- Pod management (SSH-credentialed target selection)
- Resource allocation (GPU + port selection to avoid over-allocation)
- Model→hardware mapping from a catalog (e.g. tensor-parallel-size based on GPU count)
- Remote provisioning of vLLM commands
- Web UI integration via OpenAI-compatible health-check/capability discovery

It is **not present in `packages/` at v0.74.0**. To use it, check out a pre-v0.71.0 commit (e.g. `v0.70.6`).

### pi-mom — Slack bot (removed)

*"Master of Mischief"* — a Slack bot powered by an LLM that executed bash, read/wrote files, and managed its own per-channel workspace. Per DeepWiki, key features were:

- Autonomous tool/skill creation (the bot could write its own CLI skills)
- Per-channel isolation with `MEMORY.md`, `log.jsonl`, and a `skills/` directory
- Bash + file execution with Docker or host sandbox modes
- Sequential per-channel processing queue

It is **not present in `packages/` at v0.74.0**.

### External replacement: `earendil-works/pi-chat`

The v0.74.0 root README explicitly directs users seeking Slack/chat automation to the **separate** [earendil-works/pi-chat](https://github.com/earendil-works/pi-chat) repository:

> *"Chat bot workflows: For Slack/chat automation, see earendil-works/pi-chat."*

This is an independent project outside this monorepo and outside the scope of this documentation set.

> **Note on DeepWiki:** DeepWiki's chapter list still includes sections **7. pi-pods** and **8. pi-mom**, which reflects an older snapshot of the repository. Those sections do **not** describe code that ships in `@earendil-works/pi-mono` v0.74.0.

---

## References

- Source tree: [`packages/web-ui/`](https://github.com/earendil-works/pi-mono/tree/v0.74.0/packages/web-ui) at tag `v0.74.0`
- Package README: [`packages/web-ui/README.md`](https://github.com/earendil-works/pi-mono/blob/v0.74.0/packages/web-ui/README.md)
- Package CHANGELOG: [`packages/web-ui/CHANGELOG.md`](https://github.com/earendil-works/pi-mono/blob/v0.74.0/packages/web-ui/CHANGELOG.md)
- Example app: [`packages/web-ui/example/`](https://github.com/earendil-works/pi-mono/tree/v0.74.0/packages/web-ui/example)
- Root README: [`README.md`](https://github.com/earendil-works/pi-mono/blob/v0.74.0/README.md)
- npm: [`@earendil-works/pi-web-ui`](https://www.npmjs.com/package/@earendil-works/pi-web-ui) v0.74.0
- Removal commit: [`0ed0d434` "remove mom and pods packages"](https://github.com/earendil-works/pi-mono/commit/0ed0d434) (2026-04-30)
- External: [`earendil-works/pi-chat`](https://github.com/earendil-works/pi-chat) — Slack/chat automation
- DeepWiki: [6 pi-web-ui Components](https://deepwiki.com/badlogic/pi-mono/6-pi-web-ui:-web-ui-components), [6.1 Architecture & Components](https://deepwiki.com/badlogic/pi-mono/6.1-web-ui-architecture-and-components), [6.2 Sandbox & Artifacts System](https://deepwiki.com/badlogic/pi-mono/6.2-sandbox-and-artifacts-system) — these were cross-referenced for completeness; v0.74.0 source remains the authoritative reference
