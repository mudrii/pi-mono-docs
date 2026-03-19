# Application Packages: pi-web-ui, pi-mom, pi-pods

This document covers three application-level packages in the pi-mono monorepo: the browser-based chat interface, the Slack bot, and the GPU pod management CLI.

---

## pi-web-ui: Browser Chat Interface

Package: `@mariozechner/pi-web-ui` (v0.60.0)
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

## pi-mom: Slack Bot

Package: `@mariozechner/pi-mom` (v0.60.0)
License: MIT
Source: `packages/mom/`

pi-mom is a Slack bot that delegates messages to the pi coding agent. It runs as a standalone Node.js process connecting to Slack via Socket Mode, processing messages per-channel with optional Docker sandboxing.

### Dependencies

- `@mariozechner/pi-agent-core` -- Agent state machine
- `@mariozechner/pi-ai` -- LLM provider abstraction
- `@mariozechner/pi-coding-agent` -- Session management, tool execution, skills, extensions
- `@slack/socket-mode` / `@slack/web-api` -- Slack API clients
- `@anthropic-ai/sandbox-runtime` -- Anthropic sandbox
- `croner` -- Cron scheduling for periodic events
- `diff` -- Text diffing
- `chalk` -- Console output coloring

### Architecture

```
main.ts (entry point)
├── SlackBot (slack.ts)
│   ├── SocketModeClient (Slack events)
│   ├── WebClient (Slack API)
│   ├── ChannelQueue (per-channel sequential processing)
│   └── Backfill (sync missed messages on startup)
├── AgentRunner (agent.ts)
│   ├── Agent (from pi-agent-core)
│   ├── AgentSession (from pi-coding-agent)
│   ├── SessionManager (context.jsonl persistence)
│   ├── Executor (sandbox.ts)
│   └── Tools (bash, read, write, edit, attach)
└── EventsWatcher (events.ts)
    └── Cron scheduling for periodic events
```

### Startup Flow (`src/main.ts`)

1. Parse CLI args: `mom [--sandbox=host|docker:<name>] <working-directory>` or `mom --download <channel-id>`
2. Validate environment variables (`MOM_SLACK_APP_TOKEN`, `MOM_SLACK_BOT_TOKEN`)
3. Validate sandbox (check Docker container is running if using Docker mode)
4. Create `SlackBot` with handler
5. Start events watcher for scheduled events
6. Connect to Slack

Usage modes:
- **Bot mode**: `mom [--sandbox=host|docker:<name>] <working-directory>` -- runs the interactive Slack bot
- **Download mode**: `mom --download <channel-id>` -- downloads a channel's full history as plain text

### Slack Integration (`src/slack.ts`)

The `SlackBot` class manages all Slack communication:

**Connection**: Uses Socket Mode for real-time events (no public HTTP endpoint needed). On startup:
1. Authenticates via `auth.test` to get bot user ID
2. Fetches all workspace users and channels (public, private, DMs)
3. Backfills any channels that have an existing `log.jsonl` (syncs messages that arrived while the bot was offline)
4. Records startup timestamp -- messages older than this are logged but not processed (prevents replaying old messages on reconnect)

**Event Handling**:
- `app_mention` -- Channel @mentions. Strips the @mention, logs the message, and enqueues processing. "stop" command is handled immediately (not queued) to allow interrupting running tasks.
- `message` -- All messages. Channel messages without @mention are logged (as context) but don't trigger processing. DMs always trigger processing. Bot messages, edits, and messages without text/files are ignored.

**Per-Channel Queue**: Each channel has a `ChannelQueue` that processes work items sequentially. This prevents concurrent agent runs in the same channel. Maximum queue depth is 5 for events.

**Message Logging**: All messages (user and bot) are written to `<working-dir>/<channel-id>/log.jsonl` in JSONL format with date, timestamp, user, userName, displayName, text, attachments, and isBot flag.

**Backfill**: On startup, for each channel with an existing log file, fetches up to 3 pages of recent history from the Slack API and logs any messages not already in the log file. Uses the largest existing timestamp for efficient incremental sync.

### Agent Runner (`src/agent.ts`)

The `AgentRunner` manages a persistent agent session per channel. Runners are cached (one per channel) and reuse the same `Agent` and `AgentSession` across messages.

**Session Management**:
- Uses `SessionManager` from `pi-coding-agent` with a `context.jsonl` file per channel
- `log.jsonl` is the source of truth for all channel messages
- Before each run, messages from `log.jsonl` that arrived while the bot was busy are synced into the session
- After syncing, the agent's messages are reloaded from `context.jsonl`

**System Prompt**: Rebuilt on each run with fresh data:
- Working memory (from `MEMORY.md` files -- global and channel-specific)
- Channel and user ID mappings
- Available skills (from `skills/` directories)
- Environment description (Docker vs host)
- Workspace layout documentation
- Events system documentation (immediate, one-shot, periodic)
- Log query patterns
- Available tools (bash, read, write, edit, attach)

The system prompt instructs the agent to use Slack mrkdwn formatting (not Markdown), provides channel/user ID mappings, and documents the workspace layout and events system.

**Message Processing**:
- User messages are prefixed with timestamp and username: `[2026-03-20 14:30:00+01:00] [mario]: message text`
- Image attachments (JPEG, PNG, GIF, WebP) are sent as `ImageContent` to the LLM
- Non-image attachments are referenced as file paths in `<slack_attachments>` tags

**Response Handling**:
- Tool start labels appear in the main message (`_-> label_`)
- Full tool args and results go to a thread reply
- The main Slack message accumulates content with a "..." working indicator
- On completion, the main message is replaced with only the final assistant text
- `[SILENT]` response marker deletes the message entirely (useful for periodic events with nothing to report)
- Messages exceeding Slack's 40K limit are split into multiple parts
- Usage summary (tokens, cost, context window usage) is posted to the thread

**Event Subscription**: The runner subscribes to agent events once and uses mutable `runState` to track per-run context. Events handled: `tool_execution_start`, `tool_execution_end`, `message_start`, `message_end`, `auto_compaction_start`, `auto_compaction_end`, `auto_retry_start`.

### Sandboxing (`src/sandbox.ts`)

Two sandbox modes:

**Host** (`--sandbox=host`): Commands run directly on the host machine via `sh -c`. Process trees are killed on timeout or abort using `SIGKILL` to the process group.

**Docker** (`--sandbox=docker:<container>`): Commands are wrapped in `docker exec <container> sh -c '...'` and run via the host executor. The container must be pre-created and running. Inside the container, the workspace is mounted at `/workspace`.

The `Executor` interface:

```typescript
interface Executor {
  exec(command: string, options?: ExecOptions): Promise<ExecResult>;
  getWorkspacePath(hostPath: string): string;
}
```

`ExecOptions` supports `timeout` (in seconds, with process tree kill) and `signal` (AbortSignal for cancellation). stdout/stderr are capped at 10MB each.

### Workspace Layout

```
<working-dir>/
├── MEMORY.md                    # Global memory (all channels)
├── SYSTEM.md                    # System configuration log
├── skills/                      # Global reusable CLI tools
├── events/                      # Scheduled event JSON files
└── <channel-id>/
    ├── MEMORY.md                # Channel-specific memory
    ├── log.jsonl                # Message history
    ├── context.jsonl            # LLM context (session data)
    ├── last_prompt.jsonl        # Debug: last prompt sent
    ├── attachments/             # User-shared files
    ├── scratch/                 # Agent working directory
    └── skills/                  # Channel-specific tools
```

### Events System

Events are JSON files in `<working-dir>/events/`. Three types:

- **Immediate**: `{"type": "immediate", "channelId": "...", "text": "..."}` -- triggers as soon as detected, then auto-deletes
- **One-shot**: `{"type": "one-shot", "channelId": "...", "text": "...", "at": "2026-03-20T09:00:00+01:00"}` -- triggers at a specific time, then auto-deletes
- **Periodic**: `{"type": "periodic", "channelId": "...", "text": "...", "schedule": "0 9 * * 1-5", "timezone": "Europe/Vienna"}` -- triggers on cron schedule, persists until deleted

Events are enqueued via `SlackBot.enqueueEvent()` (max 5 per channel queue). When triggered, the agent receives a message like `[EVENT:filename.json:one-shot:2026-03-20T09:00:00+01:00] Event text`.

### Skills

Skills are reusable CLI tools stored as directories with a `SKILL.md` file. Skills can be global (`workspace/skills/`) or channel-specific (`channel/skills/`). Channel skills override global ones on name collision. The system prompt lists all available skills with their descriptions.

---

## pi-pods: GPU Pod Management CLI

Package: `@mariozechner/pi` (v0.60.0)
License: MIT
Source: `packages/pods/`
Binary: `pi-pods`

pi-pods is a CLI tool for managing vLLM deployments on remote GPU pods. It handles pod provisioning, model lifecycle, and provides SSH access.

### Dependencies

- `@mariozechner/pi-agent-core` -- Used for the agent chat mode
- `chalk` -- CLI output coloring

### Configuration (`src/config.ts`)

Configuration is stored in `~/.pi/pods.json` (or `$PI_CONFIG_DIR/pods.json`):

```typescript
interface Config {
  pods: Record<string, Pod>;
  active?: string;  // Name of the active pod
}

interface Pod {
  ssh: string;              // Full SSH command (e.g., "ssh root@1.2.3.4")
  gpus: GPU[];              // Available GPUs
  models: Record<string, Model>;  // Running models
  modelsPath?: string;      // Path to model storage
  vllmVersion?: "release" | "nightly" | "gpt-oss";
}

interface GPU {
  id: number;
  name: string;
  memory: string;
}

interface Model {
  model: string;     // HuggingFace model ID
  port: number;      // vLLM API port
  gpu: number[];     // GPU IDs used
  pid: number;       // Process ID on the pod
}
```

Functions: `loadConfig()`, `saveConfig()`, `getActivePod()`, `addPod()`, `removePod()`, `setActivePod()`.

### CLI Commands (`src/cli.ts`)

**Pod Management**:
- `pi pods` -- List all configured pods (active pod marked with `*`)
- `pi pods setup <name> "<ssh>" --mount "<mount>"` -- Set up a new pod. Accepts `--vllm release|nightly|gpt-oss` to select the vLLM version to install. Optionally `--models-path <path>` to specify model storage location
- `pi pods active <name>` -- Switch the active pod
- `pi pods remove <name>` -- Remove a pod from local config
- `pi shell [<name>]` -- Open an interactive SSH shell on the active or specified pod
- `pi ssh [<name>] "<command>"` -- Run a command via SSH on the active or specified pod

**Model Management**:
- `pi start <model> --name <name> [options]` -- Start a model on the active pod. Options: `--memory <percent>` (GPU memory allocation), `--context <size>` (context window: 4k-128k), `--gpus <count>`, `--vllm <args...>` (pass raw vLLM arguments, ignores other options)
- `pi stop [<name>]` -- Stop a specific model, or all models if no name given
- `pi list` -- List running models on the active pod
- `pi logs <name>` -- Stream model logs

**Agent Chat**:
- `pi agent <name> ["<message>"...] [options]` -- Chat with a deployed model using the pi coding agent with tool support
- Options: `--continue` / `-c` (continue previous session), `--json` (JSONL output)

All model commands support `--pod <name>` to override the active pod.

### SSH Operations (`src/ssh.ts`)

Three SSH functions:

- `sshExec(sshCmd, command, options?)` -- Run command, return stdout/stderr/exitCode. Supports SSH keepalive (`ServerAliveInterval=30`, `ServerAliveCountMax=120` for up to 60 minutes)
- `sshExecStream(sshCmd, command, options?)` -- Run with streaming output to console. Supports `forceTTY` for interactive commands and `keepAlive` for long-running operations
- `scpFile(sshCmd, localPath, remotePath)` -- Copy a file to the remote pod via SCP. Parses port from the SSH command

### Model Configurations (`src/model-configs.ts`)

Predefined model configurations are stored in `src/models.json`. The `getModelConfig(modelId, gpus, requestedGpuCount)` function selects the best configuration based on:

1. Exact match on GPU count and GPU type (e.g., H200, A100)
2. Fallback to matching GPU count without type constraint
3. Returns null for unknown models (user must provide `--vllm` args)

Each config specifies vLLM command-line arguments, optional environment variables, and notes.

### Environment Variables

- `HF_TOKEN` -- HuggingFace token for downloading gated models
- `PI_API_KEY` -- API key for authenticating against vLLM endpoints
- `PI_CONFIG_DIR` -- Override config directory (default: `~/.pi`)
