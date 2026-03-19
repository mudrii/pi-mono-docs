# pi-coding-agent (`@mariozechner/pi-coding-agent`)

Package: `@mariozechner/pi-coding-agent` v0.60.0
Source: `packages/coding-agent/`
License: MIT | Node >= 20.6.0
Binary: `pi`

Pi is a minimal terminal coding harness built on `@mariozechner/pi-agent-core` and `@mariozechner/pi-ai`. It ships with four built-in tools (read, write, edit, bash), session persistence, extension system, and runs in four modes: interactive TUI, print/JSON output, RPC over stdio, and SDK for embedding.

---

## Entry Points

### CLI Entry (`src/cli.ts`)

```typescript
#!/usr/bin/env node
process.title = "pi";
import { EnvHttpProxyAgent, setGlobalDispatcher } from "undici";
import { main } from "./main.js";
setGlobalDispatcher(new EnvHttpProxyAgent());
main(process.argv.slice(2));
```

Sets up HTTP proxy support via undici, then delegates to `main()`.

### Main Function (`src/main.ts`)

The `main(args)` function orchestrates the full startup sequence:

1. Handle package commands (`install`, `remove`, `update`, `list`, `config`) and exit early if matched
2. Run migrations (auth, sessions, extensions)
3. Parse CLI args (two passes: first for extension discovery, second with extension-defined flags)
4. Create `SettingsManager`, `AuthStorage`, `ModelRegistry`, `ResourceLoader`
5. Load extensions and register extension-provided providers
6. Read piped stdin (forces print mode if present)
7. Handle `--export` (session to HTML)
8. Resolve session manager based on flags (`--session`, `--continue`, `--resume`, `--fork`, `--no-session`)
9. Resolve model (CLI flags -> scoped models -> settings default -> first available)
10. Create `AgentSession` via `createAgentSession()`
11. Dispatch to run mode: interactive, print/JSON, or RPC

### SDK Entry (`src/core/sdk.ts`)

```typescript
import { createAgentSession, SessionManager, AuthStorage, ModelRegistry } from "@mariozechner/pi-coding-agent";

const { session } = await createAgentSession({
  sessionManager: SessionManager.inMemory(),
  authStorage: AuthStorage.create(),
  modelRegistry: new ModelRegistry(authStorage),
});

await session.prompt("What files are in the current directory?");
```

---

## Run Modes

### Interactive Mode (default)

Full TUI with editor, message display, tool output, footer. Provides keyboard shortcuts, slash commands, model cycling (Ctrl+P), thinking level cycling (Shift+Tab), session tree navigation (Escape twice), and message queuing (steering via Enter, follow-up via Alt+Enter).

### Print Mode (`-p` / `--print`)

Sends prompt, streams response to stdout, exits. Also activated when stdin is piped. Supports `--mode json` for JSONL event output.

```bash
pi -p "Summarize this codebase"
cat README.md | pi -p "Summarize this text"
```

### RPC Mode (`--mode rpc`)

JSON-RPC over stdio for process integration. Uses strict LF-delimited JSONL framing. Clients must split records on `\n` only (not Unicode line separators).

```bash
pi --mode rpc
```

### MCP Mode

Not built in. Extensions can add MCP server integration.

---

## AgentSession Orchestration

`AgentSession` (`src/core/agent-session.ts`) is the core abstraction shared by all run modes. It wraps the pi-agent-core `Agent` class and adds:

### Key Responsibilities

- **Agent state access** - Proxies model, thinking level, tools, messages
- **Event subscription** - Wraps agent events with automatic session persistence
- **Session persistence** - Writes messages and state changes to JSONL via `SessionManager`
- **Model management** - Model switching, cycling through scoped models
- **Compaction** - Manual (`/compact`) and automatic (threshold + overflow recovery)
- **Auto-retry** - Exponential backoff on transient errors (configurable)
- **Bash execution** - Executes shell commands outside the agent loop
- **Extension runner** - Manages the `ExtensionRunner` lifecycle
- **Tool management** - Registers built-in + extension tools, rebuilds on changes
- **System prompt construction** - Combines base prompt + context files + skills + extension appends

### AgentSessionConfig

```typescript
interface AgentSessionConfig {
  agent: Agent;
  sessionManager: SessionManager;
  settingsManager: SettingsManager;
  cwd: string;
  scopedModels?: Array<{ model: Model<any>; thinkingLevel?: ThinkingLevel }>;
  resourceLoader: ResourceLoader;
  customTools?: ToolDefinition[];
  modelRegistry: ModelRegistry;
  initialActiveToolNames?: string[];
  baseToolsOverride?: Record<string, AgentTool>;
  extensionRunnerRef?: { current?: ExtensionRunner };
}
```

### Session-Specific Events

`AgentSessionEvent` extends the base `AgentEvent` with:

| Event | Description |
|-------|-------------|
| `auto_compaction_start` | Automatic compaction triggered (reason: `"threshold"` or `"overflow"`) |
| `auto_compaction_end` | Compaction finished (with result, aborted flag, willRetry flag) |
| `auto_retry_start` | Auto-retry attempt starting (with attempt number, delay, error) |
| `auto_retry_end` | Auto-retry finished (success/failure) |

### PromptOptions

```typescript
interface PromptOptions {
  expandPromptTemplates?: boolean;   // default: true
  images?: ImageContent[];
  streamingBehavior?: "steer" | "followUp";  // required if agent is streaming
  source?: InputSource;                       // for extension input handlers
}
```

---

## Built-in Tools

Default tool set: `read`, `bash`, `edit`, `write` (the `codingTools` array).

### Tool Registry

```typescript
// Default coding tools
const codingTools: Tool[] = [readTool, bashTool, editTool, writeTool];

// Read-only tools (for exploration)
const readOnlyTools: Tool[] = [readTool, grepTool, findTool, lsTool];

// All available tools
const allTools = { read, bash, edit, write, grep, find, ls };
```

### Tool Descriptions

| Tool | Description | Factory |
|------|-------------|---------|
| `read` | Read file contents | `createReadTool(cwd, options?)` |
| `bash` | Execute bash commands | `createBashTool(cwd, options?)` |
| `edit` | Surgical find-and-replace edits | `createEditTool(cwd)` |
| `write` | Create or overwrite files | `createWriteTool(cwd)` |
| `grep` | Search file contents for patterns (respects .gitignore) | `createGrepTool(cwd)` |
| `find` | Find files by glob pattern (respects .gitignore) | `createFindTool(cwd)` |
| `ls` | List directory contents | `createLsTool(cwd)` |

### Tool Selection

```bash
# Default tools (read, bash, edit, write)
pi

# Specific tools
pi --tools read,grep,find,ls

# No built-in tools (extension tools still work)
pi --no-tools

# Combine: disable defaults, enable specific
pi --no-tools --tools read,bash
```

### Tool Factories

Each tool has a factory function that accepts a working directory and optional configuration:

```typescript
createCodingTools(cwd: string, options?: ToolsOptions): Tool[]
createReadOnlyTools(cwd: string, options?: ToolsOptions): Tool[]
createAllTools(cwd: string, options?: ToolsOptions): Record<ToolName, Tool>
```

---

## Session Management

### JSONL Tree Structure

Sessions are stored as JSONL files. Each line is a JSON object with `type`, `id`, and `parentId` fields, forming a tree structure that enables in-place branching without creating new files.

```
~/.pi/agent/sessions/<encoded-cwd>/<uuid>.jsonl
```

The first line is always a `SessionHeader`:

```typescript
interface SessionHeader {
  type: "session";
  version?: number;    // current: 3
  id: string;
  timestamp: string;
  cwd: string;
  parentSession?: string;
}
```

### Session Entry Types

| Type | Description |
|------|-------------|
| `message` | User, assistant, or toolResult message |
| `thinking_level_change` | Records thinking level transitions |
| `model_change` | Records model switches |
| `compaction` | Summary replacing older messages |
| `branch_summary` | Summary of a branch point |
| `custom` | Extension-specific data (not sent to LLM) |
| `custom_message` | Extension messages injected into LLM context |
| `label` | User-defined bookmark on an entry |
| `session_info` | Metadata like display name |

### Branching

Every entry has an `id` and `parentId`. Branching works by appending new entries with a `parentId` pointing to any existing entry, creating a diverging path within the same file.

- `/tree` navigates the session tree in-place, allowing branch switching
- `/fork` creates a new session file from a selected branch point
- `--fork <path|id>` forks from CLI

### SessionManager

Static factory methods:

| Method | Description |
|--------|-------------|
| `SessionManager.create(cwd, sessionDir?)` | New session |
| `SessionManager.open(path, sessionDir?)` | Open existing session file |
| `SessionManager.inMemory()` | Ephemeral session (no file persistence) |
| `SessionManager.continueRecent(cwd, sessionDir?)` | Resume most recent session for cwd |
| `SessionManager.forkFrom(sourcePath, cwd, sessionDir?)` | Fork an existing session |
| `SessionManager.list(cwd, sessionDir?)` | List sessions for a project |
| `SessionManager.listAll()` | List all sessions across projects |

Key instance methods:

| Method | Description |
|--------|-------------|
| `buildSessionContext()` | Reconstruct messages, model, and thinking level from entries |
| `getSessionId()` | Session UUID |
| `getSessionFile()` | Path to JSONL file (undefined for in-memory) |
| `getBranch()` | Entries on the current branch |
| `getTree()` | Full tree structure |
| `getLeafId()` | Current leaf entry ID |
| `appendMessage(message)` | Write message entry |
| `appendModelChange(provider, modelId)` | Write model change entry |
| `appendThinkingLevelChange(level)` | Write thinking level change |
| `appendCompaction(...)` | Write compaction entry |
| `appendCustomEntry(customType, data?)` | Write extension data entry |
| `switchBranch(entryId)` | Move leaf to a different entry |

### Session CLI Flags

```bash
pi -c                  # Continue most recent session
pi -r                  # Browse and select from past sessions
pi --no-session        # Ephemeral mode (don't save)
pi --session <path>    # Use specific session file or partial UUID
pi --fork <path>       # Fork specific session into new session
pi --session-dir <dir> # Custom session storage directory
```

---

## Compaction

Long sessions can exhaust context windows. Compaction summarizes older messages while keeping recent ones.

### Manual Compaction

```
/compact                        # Default compaction
/compact <custom instructions>  # With custom guidance
```

### Automatic Compaction

Enabled by default. Two triggers:

1. **Threshold** - Proactive compaction when approaching context limit
2. **Overflow** - Recovery compaction when context overflow error occurs (retries automatically)

### Compaction Settings

```typescript
interface CompactionSettings {
  enabled?: boolean;           // default: true
  reserveTokens?: number;      // default: 16384
  keepRecentTokens?: number;   // default: 20000
}
```

The full history remains in the JSONL file; use `/tree` to revisit pre-compaction content. Extensions can customize compaction behavior via the `session_before_compact` event.

---

## Settings Management

### Two-Tier Settings

| Location | Scope |
|----------|-------|
| `~/.pi/agent/settings.json` | Global (all projects) |
| `.pi/settings.json` | Project (deep-merged over global) |

Settings are managed by `SettingsManager` which loads, merges, and provides typed access to all configuration values.

### Key Settings

```typescript
interface Settings {
  // Model defaults
  defaultProvider?: string;
  defaultModel?: string;
  defaultThinkingLevel?: "off" | "minimal" | "low" | "medium" | "high" | "xhigh";

  // Transport and delivery
  transport?: "sse" | "websocket" | "auto";        // default: "sse"
  steeringMode?: "all" | "one-at-a-time";           // default: "one-at-a-time"
  followUpMode?: "all" | "one-at-a-time";            // default: "one-at-a-time"

  // Compaction
  compaction?: CompactionSettings;

  // Retry
  retry?: RetrySettings;

  // UI
  theme?: string;
  quietStartup?: boolean;
  hideThinkingBlock?: boolean;
  doubleEscapeAction?: "fork" | "tree" | "none";     // default: "tree"
  treeFilterMode?: "default" | "no-tools" | "user-only" | "labeled-only" | "all";
  editorPaddingX?: number;
  showHardwareCursor?: boolean;

  // Terminal
  terminal?: { showImages?: boolean; clearOnShrink?: boolean };

  // Images
  images?: { autoResize?: boolean; blockImages?: boolean };

  // Thinking
  thinkingBudgets?: { minimal?: number; low?: number; medium?: number; high?: number };

  // Shell
  shellPath?: string;
  shellCommandPrefix?: string;

  // Packages and resources
  packages?: PackageSource[];
  extensions?: string[];
  skills?: string[];
  prompts?: string[];
  themes?: string[];
  enableSkillCommands?: boolean;         // default: true
  enabledModels?: string[];              // patterns for Ctrl+P cycling
  npmCommand?: string[];                 // custom npm argv for package management

  // Markdown rendering
  markdown?: { codeBlockIndent?: string };
}
```

### Retry Settings

```typescript
interface RetrySettings {
  enabled?: boolean;        // default: true
  maxRetries?: number;      // default: 3
  baseDelayMs?: number;     // default: 2000 (exponential backoff: 2s, 4s, 8s)
  maxDelayMs?: number;      // default: 60000
}
```

---

## Model Resolution

Model resolution follows a priority chain:

1. **CLI `--model`** (with optional `--provider` or `provider/id` format)
2. **Scoped models** (from `--models` flag or `enabledModels` setting)
3. **Settings default** (`defaultProvider` + `defaultModel`)
4. **First available** model from any authenticated provider

The `--model` flag supports a thinking level shorthand: `--model sonnet:high`.

Model cycling with Ctrl+P rotates through scoped models. If none are scoped, it cycles through all available models from the current provider.

### Custom Providers

Add providers via `~/.pi/agent/models.json` for APIs that speak Anthropic, OpenAI, or Google protocols. Extensions can register fully custom providers with their own OAuth flows.

---

## Extension System

Extensions are TypeScript modules that extend pi with custom tools, commands, keyboard shortcuts, event handlers, and UI components.

```typescript
export default function (pi: ExtensionAPI) {
  pi.registerTool({ name: "deploy", ... });
  pi.registerCommand("stats", { ... });
  pi.on("tool_call", async (event, ctx) => { ... });
}
```

### Extension Discovery

Extensions are loaded from:
- `~/.pi/agent/extensions/` (global)
- `.pi/extensions/` (project)
- Pi packages (npm or git)
- CLI flags (`-e`, `--extension`)

### Extension Capabilities

- Register custom tools (or replace built-in tools)
- Register slash commands
- Register keyboard shortcuts
- Handle lifecycle events (agent_start, agent_end, turn_start, turn_end, tool_call, tool_result, etc.)
- Add UI components (widgets, editors, status lines, overlays)
- Custom compaction logic
- Custom system prompt appends
- Register custom LLM providers
- Define CLI flags consumed at startup
- Session lifecycle hooks (fork, switch, tree navigation, shutdown)

### ExtensionRunner

The `ExtensionRunner` class manages the runtime lifecycle:
- Receives all agent events and dispatches to extension handlers
- Manages the `beforeToolCall`/`afterToolCall` pipeline for extension-registered tools
- Emits context transformation events
- Handles extension errors via configurable error listeners

---

## Slash Commands

### Built-in Commands

| Command | Description |
|---------|-------------|
| `/settings` | Open settings menu |
| `/model` | Switch models (selector UI) |
| `/scoped-models` | Enable/disable models for Ctrl+P cycling |
| `/login`, `/logout` | OAuth authentication |
| `/new` | Start a new session |
| `/resume` | Browse and select from past sessions |
| `/name <name>` | Set session display name |
| `/session` | Show session info and stats |
| `/tree` | Navigate session tree (switch branches) |
| `/fork` | Create a new fork from a previous message |
| `/compact [prompt]` | Manually compact context |
| `/copy` | Copy last assistant message to clipboard |
| `/export [file]` | Export session to HTML |
| `/import` | Import and resume a session from JSONL |
| `/share` | Upload as private GitHub gist |
| `/reload` | Reload keybindings, extensions, skills, prompts, and themes |
| `/hotkeys` | Show all keyboard shortcuts |
| `/changelog` | Show changelog entries |
| `/quit`, `/exit` | Quit pi |

### Extension Commands

Extensions register commands via `pi.registerCommand()`. Skills are available as `/skill:name` commands when `enableSkillCommands` is true (default).

### Prompt Templates

Reusable prompts as Markdown files in `~/.pi/agent/prompts/`, `.pi/prompts/`, or pi packages. Type `/name` to expand. Templates support `{{variable}}` placeholders.

---

## Context Files

Pi loads `AGENTS.md` (or `CLAUDE.md`) at startup from:
- `~/.pi/agent/AGENTS.md` (global)
- Parent directories (walking up from cwd)
- Current directory

All matching files are concatenated into the system prompt.

### System Prompt Override

- `.pi/SYSTEM.md` or `~/.pi/agent/SYSTEM.md` replaces the default system prompt
- `APPEND_SYSTEM.md` appends without replacing
- `--system-prompt <text>` replaces from CLI
- `--append-system-prompt <text>` appends from CLI

---

## CLI Arguments and Flags

```bash
pi [options] [@files...] [messages...]
```

### Mode Flags

| Flag | Description |
|------|-------------|
| (default) | Interactive mode |
| `-p`, `--print` | Print response and exit |
| `--mode json` | Output all events as JSON lines |
| `--mode rpc` | RPC mode for process integration |
| `--export <in> [out]` | Export session to HTML |

### Model Options

| Option | Description |
|--------|-------------|
| `--provider <name>` | Provider name |
| `--model <pattern>` | Model pattern, supports `provider/id` and `pattern:thinking` |
| `--api-key <key>` | API key (runtime only, not persisted) |
| `--thinking <level>` | Thinking level override |
| `--models <patterns>` | Comma-separated patterns for Ctrl+P cycling |
| `--list-models [search]` | List available models and exit |

### Session Options

| Option | Description |
|--------|-------------|
| `-c`, `--continue` | Continue most recent session |
| `-r`, `--resume` | Browse and select session |
| `--session <path>` | Use specific session file or partial UUID |
| `--fork <path>` | Fork specific session into new session |
| `--session-dir <dir>` | Custom session storage directory |
| `--no-session` | Ephemeral mode |

### Tool Options

| Option | Description |
|--------|-------------|
| `--tools <list>` | Comma-separated built-in tool names |
| `--no-tools` | Disable all built-in tools |

### Resource Options

| Option | Description |
|--------|-------------|
| `-e`, `--extension <source>` | Load extension (repeatable) |
| `--no-extensions` | Disable extension discovery |
| `--skill <path>` | Load skill (repeatable) |
| `--no-skills` | Disable skill discovery |
| `--prompt-template <path>` | Load prompt template (repeatable) |
| `--no-prompt-templates` | Disable prompt template discovery |
| `--theme <path>` | Load theme (repeatable) |
| `--no-themes` | Disable theme discovery |

Combine `--no-*` with explicit flags to load exactly what you need.

### File Arguments

Prefix files with `@` to include in the message:

```bash
pi @prompt.md "Answer this"
pi -p @screenshot.png "What's in this image?"
```

### Environment Variables

| Variable | Description |
|----------|-------------|
| `PI_CODING_AGENT_DIR` | Override config directory (default: `~/.pi/agent`) |
| `PI_PACKAGE_DIR` | Override package directory |
| `PI_SKIP_VERSION_CHECK` | Skip version check at startup |
| `PI_CACHE_RETENTION` | `long` for extended prompt cache |
| `VISUAL`, `EDITOR` | External editor for Ctrl+G |
| `PI_OFFLINE` | Offline mode (no network checks) |

---

## Configuration Reference

### Directory Structure

```
~/.pi/agent/
├── settings.json          # Global settings
├── auth.json              # Credentials (API keys, OAuth tokens)
├── models.json            # Custom provider/model definitions
├── keybindings.json       # Custom keyboard shortcuts
├── sessions/              # Session JSONL files (organized by cwd)
├── extensions/            # Global extensions
├── skills/                # Global skills
├── prompts/               # Global prompt templates
├── themes/                # Global themes
├── bin/                   # Managed binaries (fd, rg)
├── git/                   # Git-installed packages
├── AGENTS.md              # Global context file
├── SYSTEM.md              # Global system prompt override
└── APPEND_SYSTEM.md       # Global system prompt append

.pi/                       # Project-local config
├── settings.json          # Project settings (merged over global)
├── extensions/            # Project extensions
├── skills/                # Project skills
├── prompts/               # Project prompt templates
├── themes/                # Project themes
├── git/                   # Project git packages
├── npm/                   # Project npm packages
├── SYSTEM.md              # Project system prompt override
└── APPEND_SYSTEM.md       # Project system prompt append
```

### Migrations (`src/migrations.ts`)

On startup, `runMigrations()` handles:

1. **Auth migration** - Moves legacy `oauth.json` and `settings.json` apiKeys to `auth.json`
2. **Session migration** - Moves `.jsonl` files from `~/.pi/agent/` root to proper `sessions/<encoded-cwd>/` directories (v0.30.0 bug fix)
3. **Binary migration** - Moves fd/rg from `tools/` to `bin/`
4. **Extension system migration** - Renames `commands/` to `prompts/`, warns about deprecated `hooks/` and `tools/` directories

### Package Management

Pi packages bundle extensions, skills, prompts, and themes for sharing via npm or git.

```bash
pi install npm:@foo/pi-tools        # npm package
pi install git:github.com/user/repo # git repo
pi install npm:@foo/pi-tools@1.2.3  # pinned version
pi remove npm:@foo/pi-tools
pi update                            # update all (skips pinned)
pi list                              # list installed
pi config                            # enable/disable package resources
```

Packages install globally by default. Use `-l` for project-local installs. Package manifest in `package.json`:

```json
{
  "pi": {
    "extensions": ["./extensions"],
    "skills": ["./skills"],
    "prompts": ["./prompts"],
    "themes": ["./themes"]
  }
}
```

---

## SDK Exports (`src/index.ts`)

The package exports a comprehensive public API for programmatic usage:

**Core:** `AgentSession`, `createAgentSession`, `AuthStorage`, `ModelRegistry`, `SessionManager`, `SettingsManager`

**Tools:** `bashTool`, `editTool`, `readTool`, `writeTool`, `grepTool`, `findTool`, `lsTool`, `codingTools`, `readOnlyTools`, plus factory functions (`createBashTool`, `createEditTool`, etc.)

**Extension system:** `ExtensionRunner`, `ExtensionAPI` types, event types, tool types

**Compaction:** `compact`, `shouldCompact`, `estimateTokens`, `calculateContextTokens`, `generateSummary`

**Session:** `SessionManager`, `buildSessionContext`, `parseSessionEntries`, entry types

**UI components:** `InteractiveMode`, `runPrintMode`, `runRpcMode`, plus TUI components for building custom extension UIs

**Theme:** `Theme`, `initTheme`, `highlightCode`, `getMarkdownTheme`

**Hooks subpath export:** `@mariozechner/pi-coding-agent/hooks` provides hook types for extension development
