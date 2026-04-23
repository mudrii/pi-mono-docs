# @mariozechner/pi-web-ui

**Version:** 0.69.0 · **License:** MIT · [npm](https://www.npmjs.com/package/@mariozechner/pi-web-ui)

Web UI components for pi: chat interface, artifact panel, IndexedDB session storage, and a multi-format attachment system. Built on Lit web components.

---

## Installation

```bash
npm install @mariozechner/pi-web-ui lit @mariozechner/mini-lit
```

**Required peer dependencies:**
- `lit` ^3.3.1 — Web components framework
- `@mariozechner/mini-lit` ^0.2.0 — Custom UI utilities

---

## Quick Start

```html
<!DOCTYPE html>
<html>
<head>
  <link rel="stylesheet" href="node_modules/@mariozechner/pi-web-ui/dist/app.css">
</head>
<body>
  <pi-chat-panel id="chat"></pi-chat-panel>

  <script type="module">
    import { Agent } from "@mariozechner/pi-agent-core";
    import { getModel } from "@mariozechner/pi-ai";

    const agent = new Agent({
      initialState: {
        model: getModel("anthropic", "claude-opus-4-6"),
        systemPrompt: "You are a helpful assistant."
      }
    });

    const panel = document.getElementById("chat");
    await panel.setAgent(agent);
  </script>
</body>
</html>
```

---

## ChatPanel Component

`<pi-chat-panel>` is the top-level container that wires together the agent interface and artifact panel.

```typescript
@customElement("pi-chat-panel")
class ChatPanel extends LitElement {
  async setAgent(
    agent: Agent,
    config?: ChatPanelConfig
  ): Promise<void>;
}

interface ChatPanelConfig {
  // Called when a provider requires an API key
  onApiKeyRequired?: (provider: string) => Promise<boolean>;

  // Called just before user sends a message
  onBeforeSend?: () => void | Promise<void>;

  // Called when user clicks the cost display
  onCostClick?: () => void;

  // Called when user opens model selector
  onModelSelect?: () => void;

  // Base URL for sandboxed HTML artifact iframes
  sandboxUrlProvider?: () => string;

  // Factory for custom tools (REPL, artifacts, etc.)
  toolsFactory?: (
    agent: Agent,
    agentInterface: AgentInterface,
    artifactsPanel: ArtifactsPanel,
    runtimeProvidersFactory: () => SandboxRuntimeProvider[]
  ) => AgentTool<any>[];
}
```

**Layout behavior:**
- Viewport width > 800px: artifact panel opens side-by-side
- Viewport width ≤ 800px: artifact panel opens as overlay

---

## AgentInterface Component

`<agent-interface>` renders the conversation and message editor.

```typescript
@customElement("agent-interface")
class AgentInterface extends LitElement {
  @property() session?: Agent;
  @property() enableAttachments = true;
  @property() enableModelSelector = true;
  @property() enableThinkingSelector = true;
  @property() showThemeToggle = false;

  // Callbacks
  @property() onApiKeyRequired?: (provider: string) => Promise<boolean>;
  @property() onBeforeSend?: () => void | Promise<void>;
  @property() onBeforeToolCall?: (toolName: string, args: any) => boolean | Promise<boolean>;
  @property() onCostClick?: () => void;
  @property() onModelSelect?: () => void;

  // Programmatic control
  setInput(text: string, attachments?: Attachment[]): void;
  setAutoScroll(enabled: boolean): void;
}
```

---

## MessageEditor Component

`<message-editor>` handles user input with attachment support.

```typescript
@customElement("message-editor")
class MessageEditor extends LitElement {
  @property() value: string;
  @property() isStreaming: boolean = false;
  @property() currentModel?: Model<any>;
  @property() thinkingLevel: ThinkingLevel = "off";

  // Feature toggles
  @property() showAttachmentButton = true;
  @property() showModelSelector = true;
  @property() showThinkingSelector = true;

  // Attachment limits
  @property() maxFiles = 10;
  @property() maxFileSize = 20 * 1024 * 1024;   // 20MB
  @property() acceptedTypes = "image/*,application/pdf,...";
  @property() attachments: Attachment[] = [];

  // Callbacks
  @property() onInput?: (value: string) => void;
  @property() onSend?: (input: string, attachments: Attachment[]) => void;
  @property() onAbort?: () => void;
  @property() onModelSelect?: () => void;
  @property() onThinkingChange?: (level: ThinkingLevel) => void;
  @property() onFilesChange?: (files: Attachment[]) => void;
}
```

**Input features:**
- Multi-line textarea with auto-resize
- File drag-and-drop
- Clipboard image paste
- Model selector dropdown
- Thinking level selector (off / minimal / low / medium / high)
- Send button (idle) / Abort button (streaming)

---

## Artifact System

Artifacts are named files created by the agent and rendered in the artifact panel.

### Artifact Message Type

Pi-web-ui extends the agent message system with custom message types:

```typescript
interface ArtifactMessage {
  role: "artifact";
  action: "create" | "update" | "delete";
  filename: string;
  content?: string;
  title?: string;
  timestamp: string;
}

// TypeScript declaration merging
declare module "@mariozechner/pi-agent-core" {
  interface CustomAgentMessages {
    "artifact": ArtifactMessage;
  }
}
```

### Artifact Tool Parameters

```typescript
type ArtifactsParams = {
  command: "create" | "update" | "rewrite" | "get" | "delete" | "logs";
  filename: string;
  content?: string;
  old_str?: string;   // For "update" command (surgical edit)
  new_str?: string;   // For "update" command
}
```

### Supported Artifact Types

Detected automatically from file extension:

| Extension | Renderer | Notes |
|-----------|----------|-------|
| `.html` | Sandboxed iframe | Full JavaScript execution |
| `.svg` | Sandboxed iframe renderer | See fix note below |
| `.md`, `.markdown` | Markdown renderer | |
| `.png`, `.jpg`, `.jpeg`, `.gif`, `.webp`, `.bmp`, `.ico` | Image display | |
| `.pdf` | PDF.js viewer | |
| `.xlsx`, `.xls` | xlsx spreadsheet viewer | |
| `.docx` | docx-preview viewer | |
| `.txt` | Plain text | |
| other | Generic file display | |

> **Fix (v0.69.0):** SVG artifact previews are now rendered inside sandboxed iframes,
> preventing SVG `<script>` payloads from escaping the sandbox boundary.

### ArtifactsPanel Component

```typescript
@customElement("artifacts-panel")
class ArtifactsPanel extends LitElement {
  // Shows artifact selector (pill tabs) + active artifact renderer
  // Reconstructs artifact state from existing messages on load
}
```

### Sandbox Runtime Providers

HTML artifacts run in sandboxed iframes with access to runtime providers:

```typescript
interface SandboxRuntimeProvider {
  getData(): Record<string, any>;
  getRuntime(): (sandboxId: string) => void;
  getDescription?(): string;
}
```

**ArtifactsRuntimeProvider** — artifact CRUD from within sandbox:

```javascript
// Available as window functions inside sandboxed HTML
window.listArtifacts()                                      // → string[]
window.getArtifact(filename)                               // → any
window.createOrUpdateArtifact(filename, content, mimeType) // → void
window.deleteArtifact(filename)                            // → void
```

**AttachmentsRuntimeProvider** — access uploaded files:

```javascript
window.listAttachments()                 // → string[] (attachment IDs)
window.getAttachmentContent(attachmentId) // → string (base64 or text)
```

**ConsoleRuntimeProvider** — capture sandbox console output:

```javascript
window.getConsoleLogs() // → ConsoleLog[]

// ConsoleLog: { timestamp: number, level: "log"|"error"|"warn"|"info", args: any[] }
```

**FileDownloadRuntimeProvider** — trigger file downloads:

```javascript
window.downloadFile({ name: "data.csv", content: csvString, mimeType: "text/csv" })
```

---

## Attachment System

```typescript
interface Attachment {
  id: string;
  type: "image" | "document";
  fileName: string;
  mimeType: string;
  size: number;
  content: string;          // Base64 encoded
  extractedText?: string;   // Text extracted from documents
  preview?: string;         // Base64 image preview (for documents)
}

// Load from various sources
async function loadAttachment(
  source: string | File | Blob | ArrayBuffer,
  fileName?: string
): Promise<Attachment>;

// Convert to pi-ai ImageContent / content blocks
function convertAttachments(attachments: Attachment[]): (TextContent | ImageContent)[];
```

### Supported Attachment Formats

| Category | Formats | Processing |
|----------|---------|------------|
| Images | PNG, JPG, GIF, WebP, BMP, ICO | Base64 encoded |
| PDF | .pdf | Text extracted, first-page preview generated |
| Word | .docx | Converted to HTML via docx-preview |
| Excel | .xlsx, .xls | Parsed via xlsx library |
| PowerPoint | .pptx | Converted |
| Text/Code | .txt, .md, .json, .xml, .html, .css, .js, .ts, etc. | UTF-8 text |

Max file size: 20MB per attachment (configurable).

---

## IndexedDB Storage

### AppStorage

Central storage coordinator with specialized stores:

```typescript
class AppStorage {
  sessions: SessionsStore;
  settings: SettingsStore;
  providerKeys: ProviderKeysStore;
  customProviders: CustomProvidersStore;

  getQuotaInfo(): Promise<{ usage: number; quota: number; percent: number }>;
  requestPersistence(): Promise<boolean>;
}
```

### SessionsStore

Stores complete conversation sessions with lightweight metadata for fast listing:

```typescript
class SessionsStore {
  async save(session: SessionData): Promise<void>;
  async get(id: string): Promise<SessionData | null>;
  async getMetadata(id: string): Promise<SessionMetadata | null>;
  async getAllMetadata(): Promise<SessionMetadata[]>;
  async delete(id: string): Promise<void>;
  async updateTitle(id: string, title: string): Promise<void>;
}

interface SessionData {
  id: string;
  title: string;
  createdAt: string;         // ISO 8601
  lastModified: string;      // ISO 8601
  messages: AgentMessage[];  // Full conversation
}

interface SessionMetadata {
  id: string;
  title: string;
  createdAt: string;
  lastModified: string;
  messageCount: number;
  usage: { input: number; output: number };
}
```

**Dual-store design:**
- `sessions` store: Full session data (heavy)
- `sessions-metadata` store: Lightweight metadata only (for fast sidebar listing)
- Index on `lastModified` for sorted retrieval

### SettingsStore

Key-value store for user preferences:

```typescript
class SettingsStore {
  async get<T>(key: string): Promise<T | null>;
  async set<T>(key: string, value: T): Promise<void>;
  async delete(key: string): Promise<void>;
}
```

### ProviderKeysStore

Stores API keys for LLM providers:

```typescript
class ProviderKeysStore {
  async getKey(provider: string): Promise<string | null>;
  async setKey(provider: string, key: string): Promise<void>;
  async deleteKey(provider: string): Promise<void>;
  async getAllProviders(): Promise<string[]>;
}
```

### StorageBackend Interface

Implement to provide alternative storage backends:

```typescript
interface StorageBackend {
  get<T>(storeName: string, key: string): Promise<T | null>;
  set<T>(storeName: string, key: string, value: T): Promise<void>;
  delete(storeName: string, key: string): Promise<void>;
  keys(storeName: string, prefix?: string): Promise<string[]>;
  getAllFromIndex<T>(
    storeName: string,
    indexName: string,
    direction?: "asc" | "desc"
  ): Promise<T[]>;
  transaction(
    storeNames: string[],
    mode: "readonly" | "readwrite",
    callback: (tx: StorageTransaction) => Promise<void>
  ): Promise<void>;
  getQuotaInfo(): Promise<{ usage: number; quota: number; percent: number }>;
  requestPersistence(): Promise<boolean>;
}

interface StorageTransaction {
  get<T>(storeName: string, key: string): Promise<T | null>;
  set<T>(storeName: string, key: string, value: T): Promise<void>;
  delete(storeName: string, key: string): Promise<void>;
}
```

---

## Tool Rendering System

Register custom renderers for tool outputs in the web UI:

```typescript
function registerToolRenderer(toolName: string, renderer: ToolRenderer): void;
function getToolRenderer(toolName: string): ToolRenderer | undefined;

interface ToolRenderer {
  render(
    result: any,
    state: "inprogress" | "complete" | "error"
  ): TemplateResult;  // Lit TemplateResult
}
```

**Built-in renderers:**

| Tool | Renderer | Notes |
|------|----------|-------|
| (default) | `DefaultRenderer` | JSON display |
| `bash` | `BashRenderer` | Collapsible output with syntax highlighting |
| `calculate` | `CalculateRenderer` | Expression and result |
| `get_current_time` | `GetCurrentTimeRenderer` | Formatted time |
| `artifacts` | `ArtifactsToolRenderer` | Artifact create/update/delete indicator |

**Helper functions:**

```typescript
function renderHeader(
  state: "inprogress" | "complete" | "error",
  toolIcon: any,
  text: string | TemplateResult
): TemplateResult;

function renderCollapsibleHeader(
  state: "inprogress" | "complete" | "error",
  toolIcon: any,
  text: string | TemplateResult,
  contentRef: Ref<HTMLElement>,
  chevronRef: Ref<HTMLElement>,
  defaultExpanded?: boolean
): TemplateResult;
```

---

## Message Rendering System

Register custom message renderers per role:

```typescript
function registerMessageRenderer(role: string, renderer: MessageRenderer): void;
function getMessageRenderer(role: string): MessageRenderer | undefined;
function renderMessage(message: AgentMessage, tools?: AgentTool[]): TemplateResult;

interface MessageRenderer {
  render(message: AgentMessage, tools?: AgentTool[]): TemplateResult;
}
```

**Built-in message renderers:**

| Role | Element | Notes |
|------|---------|-------|
| `user` | `<user-message>` | Plain text |
| `user-with-attachments` | `<user-message>` | Text + attachment thumbnails |
| `assistant` | `<assistant-message>` | Text, thinking blocks, tool calls |
| `toolResult` | `<tool-message>` | Tool output with optional debug view |
| `artifact` | (hidden) | Triggers artifact panel update |

---

## Custom Tools

Pi-web-ui provides factory functions for common tools:

```typescript
// JavaScript REPL tool (runs code in sandboxed iframe)
function createJavaScriptReplTool(
  agentInterface: AgentInterface,
  artifactsPanel: ArtifactsPanel,
  runtimeProvidersFactory: () => SandboxRuntimeProvider[]
): AgentTool<any>;

// Document text extraction tool
function createExtractDocumentTool(
  agentInterface: AgentInterface
): AgentTool<any>;
```

---

## Custom Message Types

Pi-web-ui declares these custom agent message types:

```typescript
declare module "@mariozechner/pi-agent-core" {
  interface CustomAgentMessages {
    "user-with-attachments": {
      role: "user-with-attachments";
      content: string | (TextContent | ImageContent)[];
      timestamp: number;
      attachments?: Attachment[];
    };
    "artifact": {
      role: "artifact";
      action: "create" | "update" | "delete";
      filename: string;
      content?: string;
      title?: string;
      timestamp: string;
    };
  }
}
```

---

## Utilities

```typescript
// Format token counts and costs for display
formatUsage(usage: Usage): string;
formatTokenCount(n: number): string;
formatCost(cost: number): string;

// Authentication
getAuthToken(): string | null;
clearAuthToken(): void;

// Proxy configuration
applyProxyIfNeeded(agent: Agent, proxyUrl?: string): void;
createStreamFn(proxyUrl: string, authToken: string): StreamFn;
// Returns true for: anthropic, openai, google, xai, mistral, openai-codex, github-copilot (v0.58.4+)
shouldUseProxyForProvider(provider: string): boolean;

// i18n
i18n(key: string): string;
setLanguage(lang: string): void;
```

---

## Theming

Pi-web-ui uses Tailwind CSS compiled into `dist/app.css`. Import it in your HTML:

```html
<link rel="stylesheet" href="node_modules/@mariozechner/pi-web-ui/dist/app.css">
```

Light/dark theme is controlled via CSS class on the root element:

```javascript
document.documentElement.classList.toggle("dark");
```

---

## ModelSelector and SettingsDialog

### ModelSelector

`ModelSelector.open()` accepts an optional `allowedProviders` filter to restrict which providers are shown (v0.58.4+):

```typescript
import { ModelSelector } from "@mariozechner/pi-web-ui";

// Show only specific providers
ModelSelector.open({
  allowedProviders: ["anthropic", "openai", "google"],
  onSelect: (model) => agent.setModel(model)
});
```

Model search uses **subsequence-based fuzzy matching** (v0.58.4+), replacing the previous substring matching. This means typing `"clop"` matches `"claude-opus-4-6"`.

### SettingsDialog

`SettingsDialog.open()` accepts an `onClose` callback (v0.58.4+):

```typescript
import { SettingsDialog } from "@mariozechner/pi-web-ui";

SettingsDialog.open({
  agent,
  onClose: () => console.log("Settings dialog closed")
});
```

---

## LM Studio and Ollama Integration

Pi-web-ui includes first-class support for local models:

**LM Studio** — via `@lmstudio/sdk`:
- Auto-discovers models running in LM Studio
- Registers as custom provider in pi-ai

**Ollama** — via `ollama` npm package:
- Auto-discovers locally running Ollama models
- Registers as custom provider

Both are discovered automatically when the web UI loads.
