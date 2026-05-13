# pi-coding-agent (`@earendil-works/pi-coding-agent`)

| Field | Value |
|-------|-------|
| Package | `@earendil-works/pi-coding-agent` |
| Version | `0.74.0` (tag `v0.74.0`, commit `1eee081e`) |
| npm | `npm i -g @earendil-works/pi-coding-agent` |
| Binary | `pi` |
| Engine | Node `>= 20.6.0` |
| License | MIT |
| Source | `packages/coding-agent/` |
| Repo | `https://github.com/earendil-works/pi-mono` (moved from `badlogic/pi-mono` in v0.74.0) |

Pi is a terminal coding harness built on `@earendil-works/pi-agent-core` and `@earendil-works/pi-ai`. Ships with four default coding tools (`read`, `bash`, `edit`, `write`), optional read-only tools (`grep`, `find`, `ls`), JSONL session persistence with tree branching, an extension system, and four integration modes: interactive TUI, print (text/JSON), RPC over stdio, and SDK embedding.

> **v0.74.0 scope rename.** All `@mariozechner/*` packages were renamed to `@earendil-works/*` and the repo moved from `badlogic/pi-mono` to `earendil-works/pi-mono`. The `pi update --self` command in v0.73.1+ migrates an existing global install across the rename.

> **Security.** Pi packages run with full system access. Extensions execute arbitrary code, and skills can instruct the model to perform any action including running executables. Review source code before installing third-party packages. The bundled `bash`, `edit`, and `write` tools have no built-in allowlist — sandboxing is the operator's responsibility.

---

## Table of Contents

1. [Install and First Run](#install-and-first-run)
2. [CLI Reference](#cli-reference)
3. [Run Modes](#run-modes)
4. [AgentSession Lifecycle](#agentsession-lifecycle)
5. [Session Management](#session-management)
6. [Built-in Tools](#built-in-tools)
7. [Settings Management](#settings-management)
8. [Model Resolution and Thinking Levels](#model-resolution-and-thinking-levels)
9. [Extension System](#extension-system)
10. [Skills and Prompt Templates](#skills-and-prompt-templates)
11. [Theme System](#theme-system)
12. [Package Management](#package-management)
13. [HTML Export, Share, Import](#html-export-share-import)
14. [SDK and Programmatic Use](#sdk-and-programmatic-use)
15. [DeepWiki Stale-Claim Callouts](#deepwiki-stale-claim-callouts)
16. [References](#references)

---

## Install and First Run

```bash
npm i -g @earendil-works/pi-coding-agent
pi
```

First launch:

1. Detect terminal background and pick default theme (`dark` or `light`).
2. Open the provider/model menu — sign in with `/login` for Anthropic/OpenAI/Copilot or paste an API key into `auth.json` via the menu.
3. Land in an interactive session at `cwd`.

Help and version:

```bash
pi --help
pi --version
```

See `docs/quickstart.md` for the full first-run walkthrough.

---

## CLI Reference

### Source of truth

| Path:Line | What |
|-----------|------|
| `src/cli.ts` | binary entry (sets `process.title = "pi"`, installs undici proxy dispatcher, calls `main`) |
| `src/cli/args.ts:12` | `Args` interface |
| `src/cli/args.ts:55` | `isValidThinkingLevel()` |
| `src/cli/args.ts:59` | `parseArgs()` |
| `src/cli/args.ts:191` | `printHelp()` — authoritative help text |
| `src/main.ts:98` | `resolveAppMode()` — picks `interactive` / `print` / `rpc` |
| `src/main.ts:423` | `main()` — startup orchestration |
| `src/main.ts:675` | dispatches `runRpcMode(runtime)` |
| `src/main.ts:687` | constructs `InteractiveMode` |
| `src/main.ts:714` | invokes `runPrintMode(runtime, ...)` |
| `src/config.ts:425-433` | `PACKAGE_NAME`, `APP_NAME`, `CONFIG_DIR_NAME`, `ENV_AGENT_DIR`, `ENV_SESSION_DIR` constants |

### Usage

```
pi [options] [@files...] [messages...]
```

Positional arguments are messages; tokens prefixed with `@` are file attachments expanded into the first message (images become image content blocks).

### Commands (package management)

| Command | Purpose |
|---------|---------|
| `pi install <source> [-l]` | Install extension source and add to settings |
| `pi remove <source> [-l]` (alias `uninstall`) | Remove extension source from settings |
| `pi update [source|self|pi]` | Update pi binary and installed extensions (`--self` migrates scope) |
| `pi list` | List installed extensions |
| `pi config` | TUI to enable/disable package resources |
| `pi <command> --help` | Per-command help |

`-l` makes the install project-local under `.pi/`. See `docs/packages.md`.

### Mode flags

| Flag | Behaviour |
|------|-----------|
| (default) | Interactive TUI |
| `--mode <text|json|rpc>` | Output mode for `--print` / RPC dispatch |
| `--print`, `-p` | Non-interactive; print response, exit. Auto-enabled when stdin is piped |
| `--export <file> [out]` | Export session JSONL to HTML and exit |

### Model and provider

| Flag | Description |
|------|-------------|
| `--provider <name>` | Provider id (default: `google`) |
| `--model <pattern>` | Model id, supports `provider/id` and `pattern:thinking` (e.g. `sonnet:high`) |
| `--api-key <key>` | One-shot API key (not persisted) |
| `--thinking <level>` | `off` / `minimal` / `low` / `medium` / `high` / `xhigh` |
| `--models <patterns>` | Comma list of glob/fuzzy patterns for `Ctrl+P` cycling |
| `--list-models [search]` | List models (with optional fuzzy search) and exit |

### Sessions

| Flag | Description |
|------|-------------|
| `--continue`, `-c` | Resume most recent session for cwd |
| `--resume`, `-r` | Browse and pick a past session |
| `--session <path|id>` | Open exact session (path or partial UUID) |
| `--fork <path|id>` | Fork a session into a new file (mutually exclusive with `--session`, `-c`, `-r`, `--no-session`) |
| `--session-dir <dir>` | Override session storage directory |
| `--no-session` | Ephemeral, don't persist |

### Tools

| Flag | Description |
|------|-------------|
| `--tools, -t <list>` | Comma allowlist (built-in + extension + custom) |
| `--no-tools, -nt` | Disable ALL tools (built-in **and** extension) — changed in v0.68.0 |
| `--no-builtin-tools, -nbt` | Disable built-in tools only; keep extension/custom tools |

### Resources

| Flag | Description |
|------|-------------|
| `--extension, -e <path>` | Load extension (repeatable) |
| `--no-extensions, -ne` | Disable extension discovery; `-e` still loads |
| `--skill <path>` | Load skill file/dir (repeatable) |
| `--no-skills, -ns` | Disable skill discovery |
| `--prompt-template <path>` | Load prompt template file/dir (repeatable) |
| `--no-prompt-templates, -np` | Disable prompt template discovery |
| `--theme <path>` | Load theme file/dir (repeatable) |
| `--no-themes` | Disable theme discovery |
| `--no-context-files, -nc` | Disable `AGENTS.md` / `CLAUDE.md` discovery |

### System prompt

| Flag | Description |
|------|-------------|
| `--system-prompt <text>` | Replace default system prompt |
| `--append-system-prompt <text>` | Append text or file contents (repeatable) |

### Misc

| Flag | Description |
|------|-------------|
| `--verbose` | Force verbose startup (overrides `quietStartup`) |
| `--offline` | Disable startup network operations (equivalent to `PI_OFFLINE=1`) |
| `--help`, `-h` | Print help |
| `--version`, `-v` | Print version |

Extensions can register their own flags via `pi.registerFlag(...)`; they appear in `pi --help` under "Extension CLI Flags".

### Environment variables

Authoritative list lives in `src/cli/args.ts:302-342`.

| Variable | Purpose |
|----------|---------|
| `PI_CODING_AGENT_DIR` | Override config dir (default `~/.pi/agent`). Constant: `ENV_AGENT_DIR` |
| `PI_CODING_AGENT_SESSION_DIR` | Override session storage dir (overridden by `--session-dir`). v0.71.0+ |
| `PI_PACKAGE_DIR` | Override package directory (Nix/Guix store paths) |
| `PI_OFFLINE` | Disable startup network ops when `1`/`true`/`yes` — `main.ts:425` |
| `PI_SKIP_VERSION_CHECK` | Skip update check at startup |
| `PI_TELEMETRY` | Force install telemetry on/off (`1`/`0` etc.) — `core/telemetry.ts:10` |
| `PI_SHARE_VIEWER_URL` | Base URL for `/share` (default `https://pi.dev/session/`) — `config.ts:441-447` |
| `PI_CACHE_RETENTION` | `long` for extended prompt cache TTL |
| `PI_OAUTH_CALLBACK_HOST` | Bind OAuth callback servers to a custom interface (default `127.0.0.1`). v0.68.0+ |
| `PI_CODING_AGENT` | Set to `true` at startup; extensions/subprocesses detect "running inside pi". v0.67.1+ |
| `VISUAL` / `EDITOR` | External editor for `Ctrl+G` |

Provider API key env vars (verbatim from help):

`ANTHROPIC_API_KEY`, `ANTHROPIC_OAUTH_TOKEN`, `OPENAI_API_KEY`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_BASE_URL`, `AZURE_OPENAI_RESOURCE_NAME`, `AZURE_OPENAI_API_VERSION`, `AZURE_OPENAI_DEPLOYMENT_NAME_MAP`, `DEEPSEEK_API_KEY`, `GEMINI_API_KEY`, `GROQ_API_KEY`, `CEREBRAS_API_KEY`, `XAI_API_KEY`, `FIREWORKS_API_KEY`, `OPENROUTER_API_KEY`, `AI_GATEWAY_API_KEY`, `ZAI_API_KEY`, `MISTRAL_API_KEY`, `MINIMAX_API_KEY`, `MOONSHOT_API_KEY`, `OPENCODE_API_KEY`, `KIMI_API_KEY`, `CLOUDFLARE_API_KEY`, `CLOUDFLARE_ACCOUNT_ID`, `CLOUDFLARE_GATEWAY_ID`, `XIAOMI_API_KEY`, `XIAOMI_TOKEN_PLAN_CN_API_KEY`, `XIAOMI_TOKEN_PLAN_AMS_API_KEY`, `XIAOMI_TOKEN_PLAN_SGP_API_KEY`, `AWS_PROFILE`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_BEARER_TOKEN_BEDROCK`, `AWS_REGION`.

> v0.73.0 split Xiaomi MiMo into a billing provider (`api.xiaomimimo.com`, `XIAOMI_API_KEY`) and three regional Token Plan providers (`CN`, `AMS`, `SGP`).
> v0.71.0 removed the Google Gemini CLI and Antigravity providers and added Cloudflare AI Gateway, Moonshot, and Mistral Medium 3.5.

### File arguments

```
pi @prompt.md @screenshot.png "Caption this image"
```

`@`-prefixed paths are read at parse time. Images become inline image content blocks; text files are concatenated with the message. See `docs/usage.md`.

### Examples

```bash
pi                                            # interactive
pi -p "List all .ts files in src/"            # one-shot print
pi -p "Explain" < notes.md                    # piped stdin forces print
pi --mode json -p "Hello"                     # JSONL event stream
pi --mode rpc                                 # JSON-RPC over stdio
pi --tools read,grep,find,ls -p "Review src/" # read-only sandbox
pi --model openai/gpt-4o "Refactor this"
pi --model sonnet:high "Hard problem"
pi --models "github-copilot/*"                # scope Ctrl+P
pi --continue "Where did we leave off?"
pi --fork f4e2a "Try a different approach"
pi --export ~/.pi/agent/sessions/-tmp-foo/abc.jsonl output.html
```

---

## Run Modes

`resolveAppMode()` (`src/main.ts:98`) selects one of `interactive`, `print`, `rpc` from `--mode`, `--print`, and whether stdin is a TTY.

### Interactive (default)

Full TUI rendered with `@earendil-works/pi-tui`. Provides editor, scrollable message log, tool output panels, footer, keybindings, slash commands, message queueing (steer with Enter, follow-up with Alt+Enter), `Ctrl+P` model cycling, `Shift+Tab` thinking-level cycling, double-Escape session-tree navigation, hot-reloadable theme, and an autocomplete menu (`/`, `@`, `!`).

Bash mode: prefix the editor with `!` to send a one-shot shell command without invoking the LLM. Border colour changes to `bashMode` from the theme.

See `docs/usage.md` and `docs/keybindings.md`.

### Print (`-p` / `--print`)

Streams the model response to stdout and exits. Combine with `--mode json` to emit one `AgentSessionEvent` per line instead. Print mode is auto-selected when stdin is piped.

`runPrintMode(runtime, { mode })` lives in `src/modes/print/`. The JSON event schema is documented in `docs/json.md`.

### RPC (`--mode rpc`)

JSON-RPC-style line protocol over stdio for embedding pi inside other processes. **Strict LF (`\n`) framing.** Do not use Node `readline` on the client; it splits on Unicode line separators that occur in normal model output and corrupts records.

Commands accepted on stdin (`docs/rpc.md`):

| Command | Effect |
|---------|--------|
| `prompt` | Send a user message |
| `steer` | Inject a steering message into the running stream |
| `follow_up` | Queue follow-up to send after current turn |

Each event is emitted as a single JSON object terminated with `\n`.

### Export

`pi --export <session.jsonl> [output.html]` renders a session to a self-contained HTML file via `src/core/export-html/`. Templates: `template.html`, `template.css`, `template.js`; vendor scripts under `src/core/export-html/vendor/`.

---

## AgentSession Lifecycle

`AgentSession` (`src/core/agent-session.ts`) is the orchestrator shared by every run mode. It wraps the pi-agent-core `Agent` and adds session persistence, compaction, retry, model switching, extension dispatch, and tool registration.

### Construction

`createAgentSession(config: AgentSessionConfig)` (re-exported from `src/index.ts`) returns `{ session: AgentSession, runtime?, ... }`. The SDK-facing factory `createAgentSessionRuntime(factory, options)` wraps it so callers can swap sessions via `runtime.newSession()` / `runtime.switchSession()` / `runtime.fork()` without rebuilding cwd-bound services.

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
  allowedToolNames?: string[];
  baseToolsOverride?: Record<string, AgentTool>;
  extensionRunnerRef?: { current?: ExtensionRunner };
  sessionStartEvent?: SessionStartEvent;
}
```

### PromptOptions

```typescript
interface PromptOptions {
  expandPromptTemplates?: boolean;          // default: true
  images?: ImageContent[];
  streamingBehavior?: "steer" | "followUp"; // required while streaming
  source?: InputSource;                     // for extension input handlers
  preflightResult?: PreflightResult;        // skip duplicate preflight
}
```

### Events

`AgentSessionEvent` extends `AgentEvent` (from pi-agent-core) with session-aware events:

| Event | Purpose |
|-------|---------|
| `queue_update` | Pending steer/follow-up queue changed |
| `compaction_start` / `compaction_end` | Manual or auto compaction |
| `auto_retry_start` / `auto_retry_end` | Retry attempt around transient errors |
| `session_info_changed` | Display name / metadata update |
| `thinking_level_changed` | Thinking level transitioned |
| `session_shutdown` | Includes `reason` (`"quit"` / `"reload"` / `"new_session"` / `"resume"` / `"fork"`) and `targetSessionFile`. v0.68.0+ |

### Responsibilities

- Proxy agent state (model, thinking level, tools, messages) to consumers.
- Persist every message, model change, thinking-level change, and compaction into the JSONL via `SessionManager`.
- Run the steer/follow-up message queue with policy `steeringMode` / `followUpMode`.
- Drive compaction (manual `/compact` and automatic threshold + overflow recovery).
- Auto-retry on transient errors with `retry` settings.
- Execute one-shot bash via `bashExecutor` (used by `!` bash mode and `/bash` callers).
- Manage the `ExtensionRunner` lifetime and dispatch all events through it.
- Rebuild the system prompt when context files, skills, or extension appends change.

---

## Session Management

Authoritative source: `docs/sessions.md` + `docs/session-format.md`.

### Storage layout

```
~/.pi/agent/sessions/<encoded-cwd>/<uuid>.jsonl
```

`encoded-cwd` replaces `/` with `-` (e.g. `-Users-alice-projects-foo`). Override with `--session-dir` or `PI_CODING_AGENT_SESSION_DIR`, or per-project via the `sessionDir` setting.

### JSONL tree (version 3)

Every line is a JSON object with `type`, `id`, `parentId`. The first line is a `SessionHeader`:

```typescript
interface SessionHeader {
  type: "session";
  version?: number;       // current: 3
  id: string;
  timestamp: string;
  cwd: string;
  parentSession?: string; // for forks
}
```

### Entry types

| `type` | Purpose |
|--------|---------|
| `message` | User / assistant / toolResult message |
| `model_change` | Provider + model switched |
| `thinking_level_change` | Thinking level transition |
| `compaction` | Summary replacing older messages |
| `branch_summary` | Summary attached to a branch point |
| `custom` | Extension-private data (not sent to LLM) |
| `custom_message` | Extension-injected message (visible to LLM) |
| `label` | User bookmark on an entry |
| `session_info` | Display name / metadata |

### Tree and branching

Entries form a tree by `parentId`. Appending a new entry to a non-leaf entry creates a branch in the **same file** — no copying. Commands that traverse the tree:

| Command | Action |
|---------|--------|
| `/tree` | Pick any entry as the new leaf (switch branch in place). `Shift+T` toggles timestamps; `Shift+L` toggles labels; `Ctrl+O` filters by mode (`default`, `no-tools`, `user-only`, `labeled-only`, `all`) |
| `/fork` | Create a **new file** branching from a chosen user message; parent recorded in `parentSession` |
| `/clone` | Duplicate the current active branch into a new session file. v0.68.0+ |
| `/resume` | Open a past session by selecting it from a list |
| `/name <n>` | Rename current session |

See `docs/sessions.md` for the comparison table of `/tree` vs `/fork` vs `/clone`.

### SessionManager API

Static factories:

| Method | Description |
|--------|-------------|
| `SessionManager.create(cwd, sessionDir?)` | New session file |
| `SessionManager.open(path, sessionDir?)` | Open an existing JSONL |
| `SessionManager.inMemory()` | Ephemeral — no file is written |
| `SessionManager.continueRecent(cwd, sessionDir?)` | Most recent session for cwd |
| `SessionManager.forkFrom(sourcePath, cwd, sessionDir?)` | Fork into a new file |
| `SessionManager.list(cwd, sessionDir?)` | Sessions for one project |
| `SessionManager.listAll()` | Sessions across all projects |

Instance methods (selection):

| Method | Description |
|--------|-------------|
| `buildSessionContext()` | Replay branch entries to current `{messages, model, thinkingLevel}` |
| `getSessionId()` / `getSessionFile()` | UUID and JSONL path |
| `getBranch()` / `getTree()` | Current branch / full tree |
| `getLeafId()` / `switchBranch(entryId)` | Active leaf manipulation |
| `appendMessage(message)` | Persist message |
| `appendModelChange(provider, modelId)` | Persist model switch |
| `appendThinkingLevelChange(level)` | Persist thinking-level switch |
| `appendCompaction(...)` | Persist a compaction entry |
| `appendCustomEntry(customType, data?)` | Extension-private record |
| `appendLabel(...)` / `appendBranchSummary(...)` | UI annotations |

`buildSessionContext()` is the canonical replay function — both pi itself and external readers (`docs/session-format.md`) call it to derive the live agent state from JSONL.

---

## Built-in Tools

Default set: `read`, `bash`, `edit`, `write`. Read-only extras: `grep`, `find`, `ls` (off by default; enable with `--tools`).

`ToolName` and helpers live in `src/core/tools/index.ts`:

```typescript
type ToolName = "read" | "bash" | "edit" | "write" | "grep" | "find" | "ls";
const allToolNames: Set<ToolName>;
```

### Tools

| Name | Purpose | Factory |
|------|---------|---------|
| `read` | Read a file (line ranges, image decoding, binary detection) | `createReadTool(cwd, options?)` |
| `bash` | Spawn a shell command with output streaming and timeout | `createBashTool(cwd, options?)` |
| `edit` | Surgical find/replace with full-string-required uniqueness check | `createEditTool(cwd, options?)` |
| `write` | Create or overwrite a file (read-before-write enforced for existing files) | `createWriteTool(cwd, options?)` |
| `grep` | ripgrep-backed content search; respects `.gitignore` | `createGrepTool(cwd, options?)` |
| `find` | fd/glob file finder; respects `.gitignore` | `createFindTool(cwd, options?)` |
| `ls` | Directory listing | `createLsTool(cwd, options?)` |

Each factory has a paired `*Definition` factory (`createReadToolDefinition` etc.) for use with `customTools` and the `ToolDefinition<Input, Details>` extension API.

### Helper aggregates

```typescript
createCodingTools(cwd, options?)        // [read, bash, edit, write]
createReadOnlyTools(cwd, options?)      // [read, grep, find, ls]
createAllTools(cwd, options?)           // Record<ToolName, Tool>
createCodingToolDefinitions(cwd, options?)
createReadOnlyToolDefinitions(cwd, options?)
createAllToolDefinitions(cwd, options?)
createTool(name, cwd, options?)         // single tool by name
createToolDefinition(name, cwd, options?)
```

`ToolsOptions` is a record of per-tool option types (`ReadToolOptions`, `BashToolOptions`, etc.).

### Tool selection (CLI)

```bash
pi                                # default coding tools
pi --tools read,grep,find,ls      # explicit allowlist (read-only sandbox)
pi --no-tools                     # disable ALL (incl. extension/custom) — v0.68.0 change
pi --no-builtin-tools             # disable built-ins; keep extension/custom
pi --no-tools --tools read,bash   # start empty, allowlist back
```

> **v0.68.0 breaking changes.**
> - `tools` on `createAgentSession()` now takes **string names**, not `Tool[]` instances. Pass `["read", "bash"]`.
> - Prebuilt tool **instance** exports (`readTool`, `bashTool`, `editTool`, `writeTool`, `grepTool`, `findTool`, `lsTool`, `readOnlyTools`, `codingTools`, and their `*ToolDefinition` siblings) have been **removed**. Use factories (`createReadTool(cwd)` etc.) or pass tool names.
> - `--no-tools` now disables built-ins **and** extension/custom tools (previously only built-ins).
> - `DefaultResourceLoader`, `loadProjectContextFiles()`, and `loadSkills()` require an explicit `cwd`.

### Bash details

`createBashTool(cwd, { spawnHook?, operations? })` lets extensions intercept spawn (replace shell, inject env, sandbox) without re-implementing the tool. Bash output streams incrementally to the TUI in v0.73.0+; long-running output is compacted in the rendered transcript.

### Edit and Write details

Edits require the `old_string` to match **exactly once** in the file (no fuzzy match). Writes require a `read` first for any file that already exists. Both go through `withFileMutationQueue(...)` (`src/core/tools/file-mutation-queue.ts`) so concurrent writes to the same path serialise.

---

## Settings Management

Two-tier deep-merged JSON.

| Path | Scope |
|------|-------|
| `~/.pi/agent/settings.json` | Global |
| `.pi/settings.json` | Project (deep-merge wins over global) |

`SettingsManager` (`src/core/settings-manager.ts`) loads both, watches for changes, and exposes typed accessors. Per-key edits go via the `/settings` UI.

### Shape (selected keys — see `docs/settings.md` for the full list)

```typescript
interface Settings {
  // Model & thinking
  defaultProvider?: string;
  defaultModel?: string;
  defaultThinkingLevel?: "off" | "minimal" | "low" | "medium" | "high" | "xhigh";
  thinkingBudgets?: { minimal?: number; low?: number; medium?: number; high?: number };
  enabledModels?: string[];                    // Ctrl+P scope

  // Compaction
  compaction?: {
    enabled?: boolean;        // default true
    reserveTokens?: number;   // default 16384
    keepRecentTokens?: number;// default 20000
  };

  // Branch summary
  branchSummary?: {
    enabled?: boolean;
    model?: string;
    maxChars?: number;
  };

  // Retry (with per-provider override)
  retry?: {
    enabled?: boolean;        // default true
    maxRetries?: number;      // default 3
    baseDelayMs?: number;     // default 2000
    maxDelayMs?: number;      // default 60000
    provider?: Record<string, Partial<RetrySettings>>;
  };

  // Message delivery
  steeringMode?: "all" | "one-at-a-time";      // default one-at-a-time
  followUpMode?: "all" | "one-at-a-time";

  // UI
  theme?: string;
  quietStartup?: boolean;
  hideThinkingBlock?: boolean;
  doubleEscapeAction?: "fork" | "tree" | "none"; // default "tree"
  treeFilterMode?: "default" | "no-tools" | "user-only" | "labeled-only" | "all";
  editorPaddingX?: number;
  showHardwareCursor?: boolean;
  autocompleteMaxVisible?: number;
  markdown?: { codeBlockIndent?: string };

  // Warnings
  warnings?: { anthropicExtraUsage?: boolean };

  // Terminal & images
  terminal?: { showImages?: boolean; clearOnShrink?: boolean; imageWidthCells?: number };
  images?: { autoResize?: boolean; blockImages?: boolean };

  // Shell / packages
  shellPath?: string;
  shellCommandPrefix?: string;
  npmCommand?: string[];

  // Sessions
  sessionDir?: string;

  // Resources
  packages?: PackageSource[];
  extensions?: string[];
  skills?: string[];
  prompts?: string[];
  themes?: string[];
  enableSkillCommands?: boolean;               // default true
  enableInstallTelemetry?: boolean;            // overridden by PI_TELEMETRY
}
```

`settings.json` accepts JSONC (comments and trailing commas) in v0.73.1+.

### Auth and custom providers

| File | Purpose |
|------|---------|
| `~/.pi/agent/auth.json` | API keys, OAuth tokens (managed by `AuthStorage`) |
| `~/.pi/agent/models.json` | Custom provider/model definitions (JSONC, v0.73.1+) |
| `~/.pi/agent/keybindings.json` | Keybinding overrides |

OAuth login in v0.73.1+ shows a provider selection menu when more than one OAuth-capable provider is available.

---

## Model Resolution and Thinking Levels

Resolution order (`src/core/model-resolver.ts` + `model-registry.ts`):

1. **CLI `--model`** (optionally `--provider`, or `provider/id` syntax, optional `:thinking` suffix).
2. **Scoped models** — `--models <patterns>` or `enabledModels` setting; first match wins.
3. **Settings default** — `defaultProvider` + `defaultModel`.
4. **First available** model from any authenticated provider.

If none match, pi opens the model selector at startup.

### Thinking levels

Six discrete levels: `off`, `minimal`, `low`, `medium`, `high`, `xhigh` (`src/cli/args.ts:53`). Validated by `isValidThinkingLevel()`. Cycle live with `Shift+Tab`.

### `thinkingLevelMap` (v0.72.0+)

Models declare a per-model mapping from level → provider-native value:

```jsonc
// ~/.pi/agent/models.json
{
  "providers": [/* … */],
  "models": [
    {
      "id": "my-thinking-model",
      "provider": "openai-compat",
      "thinkingLevelMap": {
        "off":     { "kind": "none" },
        "minimal": { "kind": "effort", "value": "low" },
        "low":     { "kind": "effort", "value": "low" },
        "medium":  { "kind": "effort", "value": "medium" },
        "high":    { "kind": "effort", "value": "high" },
        "xhigh":   { "kind": "effort", "value": "high" }
      }
    }
  ]
}
```

> v0.72.0 replaced the older `compat.reasoningEffortMap` shape with `thinkingLevelMap`. Both `Model` and `modelOverrides` accept it (`src/core/model-registry.ts:145,165,301,597,895,944` and `src/core/extensions/types.ts:1363`).

### Custom providers and models

`~/.pi/agent/models.json` (JSONC in v0.73.1+) declares custom providers and per-model overrides. Each model can set its own `baseUrl` (v0.72.0+). Values can be literal strings, `ENV:VAR_NAME`, or `!command args` (shell command output). Full schema in `docs/models.md`.

Extensions register fully custom providers (including their own OAuth flows) via `pi.registerProvider(...)`.

### `--list-models`

```bash
pi --list-models             # all available, grouped by provider
pi --list-models sonnet      # fuzzy search
```

---

## Extension System

Extensions are TypeScript modules loaded via `jiti` (no build step). Source lives in `src/core/extensions/`. Full docs in `docs/extensions.md` and `docs/sdk.md`.

### Discovery

Loaded from:

- `~/.pi/agent/extensions/` (global)
- `.pi/extensions/` (project)
- Pi packages (`extensions/` directory or `pi.extensions` in `package.json`)
- `extensions` array in `settings.json`
- CLI: `-e <path>` (repeatable)

Disable discovery with `--no-extensions`; explicit `-e` flags still load.

### Shape

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox"; // v0.69.0+: typebox 1.x (not @sinclair/typebox)

export default function (pi: ExtensionAPI) {
  pi.registerTool({
    name: "deploy",
    description: "Deploy to staging",
    parameters: Type.Object({ env: Type.String() }),
    execute: async (_callId, { env }, ctx) => ({
      content: [{ type: "text", text: `Deploying to ${env}` }],
      isError: false,
    }),
  });

  pi.registerCommand("stats", { description: "Show stats", run: async (ctx) => { /* … */ } });
  pi.registerFlag({ name: "plan", type: "boolean", description: "Plan mode" });

  pi.on("tool_call", async (event, ctx) => { /* … */ });
}
```

Factories may be **async** (v0.71+); pi awaits the default export before binding events.

### Capabilities

- `registerTool(...)` — custom tools (or replace built-ins via name collision).
- `registerCommand(name, { ... })` — slash command.
- `registerFlag({ name, type, description })` — adds a CLI flag visible in `pi --help`.
- `registerShortcut({ key, action })` — keyboard binding.
- `registerProvider(...)` — fully custom LLM provider.
- `pi.on(event, handler)` — agent + session lifecycle events.
- `ctx.ui.*` — write to TUI: `setWorkingIndicator`, `addAutocompleteProvider` (v0.69.0+), widgets, overlays.
- Tool results may return `{ terminate: true }` to end the current tool batch without an automatic follow-up LLM call (v0.69.0+).

### Lifecycle events

`agent_start`, `before_agent_start` (with `systemPromptOptions: BuildSystemPromptOptions` since v0.68.0), `agent_end`, `turn_start`, `turn_end`, `tool_call`, `tool_result`, `session_before_compact`, `session_shutdown` (with `reason` + `targetSessionFile`, v0.68.0+), `session_fork`, `session_switch`, plus session-aware events from `AgentSession`.

### ExtensionRunner

`src/core/extensions/extension-runner.ts` owns the lifecycle: it dispatches all events to registered handlers, manages `beforeToolCall`/`afterToolCall` for extension-registered tools, emits context-transformation events, and routes errors through configurable error listeners.

### defineTool() (v0.65.0+)

Standalone tool definitions with full TS parameter inference, usable outside an extension:

```typescript
import { defineTool } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox";

const greet = defineTool({
  name: "greet",
  description: "Greet someone",
  parameters: Type.Object({ name: Type.String() }),
  execute: async (_id, { name }) => ({
    content: [{ type: "text", text: `Hello, ${name}!` }],
    isError: false,
  }),
});
```

> **v0.69.0 import change.** Import `Type` from `typebox` (1.x). `@sinclair/typebox` is still aliased at the root for legacy code, but `@sinclair/typebox/compiler` is no longer shimmed. The new typebox works in eval-restricted runtimes (Cloudflare Workers).

---

## Skills and Prompt Templates

### Skills (Agent Skills standard)

Markdown files with YAML frontmatter that the model loads on demand. Locations:

- `~/.pi/agent/skills/` (global)
- `.pi/skills/` (project)
- `~/.agents/skills/` and `.agents/skills/` (cross-tool)
- Pi packages (`skills/` directory or `pi.skills`)
- `skills` array in `settings.json`
- CLI: `--skill <path>` (repeatable)

Each skill becomes a slash command `/skill:<name>` when `enableSkillCommands` is true (default). Disable discovery with `--no-skills`. Full spec in `docs/skills.md`.

### Prompt templates

Reusable user prompts. Markdown with optional frontmatter (`description`, `argument-hint`). Type `/<name>` to expand. Positional args available as `$1`, `$2`, `$@`, `${@:N}`.

Locations parallel skills: `~/.pi/agent/prompts/`, `.pi/prompts/`, package `prompts/` / `pi.prompts`, settings `prompts` array, `--prompt-template <path>`. Disable with `--no-prompt-templates`. Full spec in `docs/prompt-templates.md`.

### Context files

Pi automatically concatenates these into the system prompt unless `--no-context-files` is set:

- `~/.pi/agent/AGENTS.md` (global)
- `AGENTS.md` or `CLAUDE.md` walking up from cwd
- `AGENTS.md` or `CLAUDE.md` in cwd

System prompt override layers (highest wins for replacement):

| Layer | Effect |
|-------|--------|
| `~/.pi/agent/SYSTEM.md`, `.pi/SYSTEM.md` | Replace default |
| `~/.pi/agent/APPEND_SYSTEM.md`, `.pi/APPEND_SYSTEM.md` | Append |
| `--system-prompt <text>` | Replace from CLI |
| `--append-system-prompt <text>` | Append from CLI (repeatable) |

---

## Theme System

Authoritative source: `docs/themes.md`.

Themes are JSON files (with `$schema` pointer for editor validation) defining 51 required color tokens plus optional `export` colors for HTML export. Colors accept hex (`"#ff0000"`), 256-color index (`39`), variable references (resolved via the `vars` block), or `""` for terminal default.

### Locations

| Source | Path |
|--------|------|
| Built-in | `dark`, `light` |
| Global | `~/.pi/agent/themes/*.json` |
| Project | `.pi/themes/*.json` |
| Packages | `themes/` dir or `pi.themes` in `package.json` |
| Settings | `themes` array |
| CLI | `--theme <path>` (repeatable) |

Disable discovery with `--no-themes`. The active theme is selected via `/settings` or `settings.theme`. **Hot reload**: editing the active custom theme file reloads it in place.

### Token groups (51 required)

| Group | Count | Tokens |
|-------|-------|--------|
| Core UI | 11 | `accent`, `border`, `borderAccent`, `borderMuted`, `success`, `error`, `warning`, `muted`, `dim`, `text`, `thinkingText` |
| Backgrounds & content | 11 | `selectedBg`, `userMessageBg`, `userMessageText`, `customMessageBg`, `customMessageText`, `customMessageLabel`, `toolPendingBg`, `toolSuccessBg`, `toolErrorBg`, `toolTitle`, `toolOutput` |
| Markdown | 10 | `mdHeading`, `mdLink`, `mdLinkUrl`, `mdCode`, `mdCodeBlock`, `mdCodeBlockBorder`, `mdQuote`, `mdQuoteBorder`, `mdHr`, `mdListBullet` |
| Tool diffs | 3 | `toolDiffAdded`, `toolDiffRemoved`, `toolDiffContext` |
| Syntax highlighting | 9 | `syntaxComment`, `syntaxKeyword`, `syntaxFunction`, `syntaxVariable`, `syntaxString`, `syntaxNumber`, `syntaxType`, `syntaxOperator`, `syntaxPunctuation` |
| Thinking levels | 6 | `thinkingOff`, `thinkingMinimal`, `thinkingLow`, `thinkingMedium`, `thinkingHigh`, `thinkingXhigh` |
| Bash mode | 1 | `bashMode` |

Optional `export: { pageBg, cardBg, infoBg }` overrides HTML export colours (derived from `userMessageBg` if absent).

---

## Package Management

Pi packages bundle extensions, skills, prompts, and themes for sharing. Source: `src/core/package-manager.ts` and `docs/packages.md`.

### Sources

| Form | Example |
|------|---------|
| npm | `npm:@earendil-works/pi-plan-mode` (optional `@version`) |
| git | `git:github.com/user/repo` or `git+https://…` |
| https | `https://example.com/pi-tools.tar.gz` |
| ssh | `ssh://git@github.com/user/repo.git` |
| local | `file:/abs/path` or relative path |

### Commands

```bash
pi install npm:@foo/pi-tools             # global install
pi install npm:@foo/pi-tools@1.2.3       # pinned version
pi install npm:@foo/pi-tools -l          # project-local under .pi/
pi install git:github.com/user/repo
pi remove npm:@foo/pi-tools
pi update                                 # update all (skips pinned)
pi update self                            # update pi itself (v0.70.x+)
pi update pi                              # alias for `update self`
pi list
pi config                                 # TUI to toggle resources per package
```

### Layout

Installed packages live under:

- `~/.pi/agent/git/` and `~/.pi/agent/npm/` (global)
- `.pi/git/` and `.pi/npm/` (project, with `-l`)

A package's `package.json` declares resources:

```jsonc
{
  "pi": {
    "extensions": ["./extensions"],
    "skills": ["./skills"],
    "prompts": ["./prompts"],
    "themes": ["./themes"]
  }
}
```

### Telemetry

Install telemetry posts an anonymous record to `https://pi.dev/api/report-install` when a package is installed for the first time. Control via the `enableInstallTelemetry` setting or `PI_TELEMETRY` env var.

### Offline

`pi --offline` or `PI_OFFLINE=1` disables startup network ops (version check, telemetry, package update fetches).

---

## HTML Export, Share, Import

### `/export` and `pi --export`

Renders a session JSONL to a self-contained HTML file. Templates: `src/core/export-html/template.{html,css,js}` plus vendor scripts. Theme colours from the optional `export` block (or derived from `userMessageBg`).

```bash
pi --export ~/.pi/agent/sessions/-tmp-foo/abc.jsonl
pi --export session.jsonl output.html
```

In interactive mode: `/export [file]` writes to the given path or a derived name.

### `/share`

Uploads the rendered HTML as a **private GitHub gist** and returns a viewer URL. Requires gh auth or `GITHUB_TOKEN`. Default viewer base URL: `https://pi.dev/session/<gist-id>`; override with `PI_SHARE_VIEWER_URL`.

### `/import`

Resume from a JSONL file:

```
/import /path/to/session.jsonl
```

Pi rebuilds the session context with `buildSessionContext()` and continues from the leaf entry. Useful for restoring sessions exported from other machines.

### `/copy`

Copies the last assistant message to the clipboard (`@mariozechner/clipboard` optional dep; falls back to OSC 52).

---

## SDK and Programmatic Use

Authoritative source: `docs/sdk.md`. Public surface lives in `src/index.ts`; hooks subpath is `@earendil-works/pi-coding-agent/hooks`.

### Quick start

```typescript
import {
  createAgentSession,
  AuthStorage,
  ModelRegistry,
  SessionManager,
} from "@earendil-works/pi-coding-agent";

const authStorage = AuthStorage.create();
const modelRegistry = ModelRegistry.create(authStorage);

const { session } = await createAgentSession({
  cwd: process.cwd(),
  sessionManager: SessionManager.inMemory(),
  authStorage,
  modelRegistry,
});

await session.prompt("What files are in the current directory?");
```

### AgentSessionRuntime (v0.65.0+)

For SDK callers that need to swap sessions (start new, fork, switch), use the runtime wrapper. It captures process-global inputs once and rebuilds all cwd-bound services on each switch — startup, `/new`, `/resume`, `/fork`, and `/import` all use the same factory.

```typescript
import {
  type CreateAgentSessionRuntimeFactory,
  createAgentSessionFromServices,
  createAgentSessionRuntime,
  createAgentSessionServices,
  getAgentDir,
  SessionManager,
} from "@earendil-works/pi-coding-agent";

const createRuntime: CreateAgentSessionRuntimeFactory = async ({
  cwd, sessionManager, sessionStartEvent,
}) => {
  const services = await createAgentSessionServices({ cwd });
  return {
    ...(await createAgentSessionFromServices({ services, sessionManager, sessionStartEvent })),
    services,
    diagnostics: services.diagnostics,
  };
};

const runtime = await createAgentSessionRuntime(createRuntime, {
  cwd: process.cwd(),
  agentDir: getAgentDir(),
  sessionManager: SessionManager.create(process.cwd()),
});

await runtime.newSession();
await runtime.switchSession("/path/to/session.jsonl");
await runtime.fork("entry-id");
// runtime.session is the new live session after each call.
```

> **v0.69.0 breaking change.** After `ctx.newSession()` / `ctx.fork()` / `ctx.switchSession()`, **pre-switch** captured extension `pi` and `ctx` references throw on use. Pass a `withSession` callback to do post-switch work:
> ```typescript
> await ctx.newSession({
>   withSession: async (newCtx) => { await newCtx.pi.sendUserMessage("hello"); }
> });
> ```

### Public exports

Selected types and functions exported from `src/index.ts`:

| Area | Exports |
|------|---------|
| Core | `AgentSession`, `createAgentSession`, `AgentSessionRuntime`, `createAgentSessionRuntime`, `createAgentSessionFromServices`, `createAgentSessionServices` |
| Auth | `AuthStorage` |
| Models | `ModelRegistry`, `getAgentDir` |
| Sessions | `SessionManager`, `buildSessionContext`, `parseSessionEntries`, entry types |
| Settings | `SettingsManager`, `Settings` interface |
| Tools | `createReadTool`, `createBashTool`, `createEditTool`, `createWriteTool`, `createGrepTool`, `createFindTool`, `createLsTool`, `createCodingTools`, `createReadOnlyTools`, `createAllTools`, plus matching `*ToolDefinition` factories and `withFileMutationQueue` |
| Tool helpers | `defineTool`, `wrapRegisteredTools` |
| Compaction | `compact`, `shouldCompact`, `estimateTokens`, `calculateContextTokens`, `generateSummary` |
| Extensions | `ExtensionRunner`, `ExtensionAPI`, event/tool type bundles |
| Modes | `InteractiveMode`, `runPrintMode`, `runRpcMode`, `RpcClient` |
| UI | TUI components for building extension UIs |
| Theme | `Theme`, `initTheme`, `highlightCode`, `getMarkdownTheme` |
| Hooks | `@earendil-works/pi-coding-agent/hooks` subpath (hook type definitions) |

### Diagnostics

Arg parsing, service creation, session option resolution, and resource loading return structured `info`/`warning`/`error` diagnostics — the application layer chooses presentation and exit behaviour. Unknown single-dash flags emit an `error` diagnostic instead of being silently ignored (v0.65.0+).

---

## Directory and File Layout

```
~/.pi/agent/
├── settings.json          # Global settings (JSONC since v0.73.1)
├── auth.json              # API keys, OAuth tokens
├── models.json            # Custom providers/models (JSONC since v0.73.1)
├── keybindings.json       # Keybinding overrides
├── sessions/<cwd>/<uuid>.jsonl
├── extensions/            # Global extensions
├── skills/                # Global skills
├── prompts/               # Global prompt templates
├── themes/                # Global themes
├── bin/                   # Managed binaries (fd, rg)
├── git/                   # Git-installed packages
├── npm/                   # npm-installed packages
├── AGENTS.md
├── SYSTEM.md
└── APPEND_SYSTEM.md

.pi/                       # Project-local equivalents
├── settings.json
├── extensions/  skills/  prompts/  themes/
├── git/  npm/
├── SYSTEM.md
└── APPEND_SYSTEM.md
```

Path override env vars: `PI_CODING_AGENT_DIR`, `PI_CODING_AGENT_SESSION_DIR`, `PI_PACKAGE_DIR`.

### Startup migrations (`src/migrations.ts`)

`runMigrations()` runs on every start and handles:

1. Auth — legacy `oauth.json` and `settings.json` API keys move into `auth.json`.
2. Sessions — `.jsonl` files at the root of `~/.pi/agent/` move into `sessions/<encoded-cwd>/`.
3. Binaries — `fd`/`rg` move from `tools/` to `bin/`.
4. Extensions — `commands/` renames to `prompts/`; warns on deprecated `hooks/` and `tools/` dirs.

---

## Version Notes

Authoritative changelog: `CHANGELOG.md`. Selected highlights for v0.65.0 → v0.74.0:

| Version | Headline |
|---------|----------|
| **v0.74.0** | Scope rename: `@mariozechner/*` → `@earendil-works/*`; repo moved to `earendil-works/pi-mono` |
| **v0.73.1** | `pi update --self` migrates across the scope rename; OAuth login selection menu; JSONC for `models.json` and `settings.json` |
| **v0.73.0** | Xiaomi MiMo split (billing + regional token plan); incremental bash output streaming; compact read rendering |
| **v0.72.0** | `thinkingLevelMap` replaces `compat.reasoningEffortMap`; per-model `baseUrl` on custom providers; `shouldStopAfterTurn` extension hook |
| **v0.71.0** | Removed Google Gemini CLI / Antigravity providers; added Cloudflare AI Gateway, Moonshot, Mistral Medium 3.5; `PI_CODING_AGENT_SESSION_DIR` env var |
| **v0.70.x** | Cloudflare Workers AI provider; `pi update --self`; `pi.dev` user agent |
| **v0.69.0** | Migrated to typebox 1.x; `terminate: true` tool results; OSC 9;4 progress indicators; `ctx.ui.addAutocompleteProvider` |
| **v0.68.0** | Tool name allowlist (string[] not `Tool[]`); `--no-tools` = ALL; `/clone` command; configurable working indicator; `systemPromptOptions` in `before_agent_start`; `session_shutdown` reasons; `PI_OAUTH_CALLBACK_HOST` |
| **v0.67.x** | `AWS_BEARER_TOKEN_BEDROCK`; `PI_CODING_AGENT` self-detection env var |
| **v0.66.x** | Earendil announcement (now `/dementedelves`); Anthropic subscription auth warning; stream-error retry fix |
| **v0.65.0** | `AgentSessionRuntime`; `defineTool()`; unified diagnostics |

---

## DeepWiki Stale-Claim Callouts

The DeepWiki "section 4" pages (4.1–4.13) currently reflect a pre-v0.74.0 snapshot. Treat the following as stale where they contradict this doc or `CHANGELOG.md`:

| DeepWiki claim | Actual (v0.74.0) |
|---|---|
| Package scope `@mariozechner/pi-coding-agent` | `@earendil-works/pi-coding-agent` (renamed v0.74.0) |
| Repo `badlogic/pi-mono` | `earendil-works/pi-mono` |
| Prebuilt tool instance exports (`readTool`, `bashTool`, `codingTools`, etc.) | Removed in v0.68.0; use factories (`createReadTool(cwd)`) |
| `--no-tools` only disables built-ins | Disables ALL tools (built-in + extension + custom) since v0.68.0 |
| `tools` on `createAgentSession()` takes `Tool[]` | Takes `string[]` tool names since v0.68.0 |
| `import { Type } from "@sinclair/typebox"` in extensions | Import from `typebox` since v0.69.0 |
| `compat.reasoningEffortMap` for thinking levels | Replaced by `thinkingLevelMap` in v0.72.0 |
| Google Gemini CLI / Antigravity providers built in | Removed in v0.71.0 |
| Single Xiaomi MiMo provider | Split into billing + three regional token-plan providers in v0.73.0 |
| Mentions of `AgentSession.newSession() / .switchSession()` | Moved to `AgentSessionRuntime` in v0.65.0; old methods removed |
| `ctx.newSession()` returns active session synchronously | Pre-switch `pi`/`ctx` references throw post-switch since v0.69.0; use `withSession` callback |
| No `/clone` command | Added in v0.68.0 |
| No `/share` or `PI_SHARE_VIEWER_URL` | Both present (private GitHub gist + viewer override) |
| Settings file is plain JSON only | JSONC supported in v0.73.1+ |
| `models.json` is plain JSON only | JSONC supported in v0.73.1+ |
| OAuth login is single-provider | v0.73.1+ shows a provider selection menu |

Always cross-check against `packages/coding-agent/CHANGELOG.md` and the first-party `docs/` tree.

---

## References

### First-party docs (`packages/coding-agent/docs/`)

| File | Topic |
|------|-------|
| `docs/index.md` | Navigation |
| `docs/quickstart.md` | Install, auth, first session |
| `docs/usage.md` | Slash commands, message queue, sessions, context files, CLI cheat sheet |
| `docs/sessions.md` | `/resume`, `/tree`, `/fork`, `/clone`, filter modes, branch summaries |
| `docs/session-format.md` | JSONL v1/v2/v3 layout, entry types, `buildSessionContext()` |
| `docs/settings.md` | Every setting key (model, UI, warnings, compaction, branch summary, retry, delivery, terminal, shell, sessions, resources) |
| `docs/extensions.md` | Extension API, typebox imports, async factories |
| `docs/sdk.md` | `createAgentSession`, `AgentSessionRuntime`, `PromptOptions`, preflight |
| `docs/rpc.md` | LF-framed JSONL protocol (`prompt`, `steer`, `follow_up`) |
| `docs/json.md` | `AgentSessionEvent` schema for `--mode json` |
| `docs/skills.md` | Locations, `/skill:name`, frontmatter |
| `docs/prompt-templates.md` | Locations, frontmatter, positional args |
| `docs/models.md` | Custom providers/models, value resolvers, `thinkingLevelMap` |
| `docs/custom-provider.md` | Per-model `baseUrl`, Anthropic/OpenAI/Google compat |
| `docs/providers.md` | Built-in provider reference |
| `docs/packages.md` | Install/remove/list/update commands, package sources |
| `docs/compaction.md` | Threshold + overflow recovery, settings |
| `docs/themes.md` | Tokens, theme schema, hot reload |
| `docs/keybindings.md` | Default bindings, override file |
| `docs/tui.md` | Interactive mode layout |
| `docs/development.md` | Local development / build |
| Platform | `windows.md`, `termux.md`, `tmux.md`, `terminal-setup.md`, `shell-aliases.md` |

### Cross-references in this docs tree

| File | Covers |
|------|--------|
| `01-architecture-overview.md` | How pi-coding-agent fits into pi-mono |
| `02-pi-ai-llm-abstraction.md` | The `Model` / provider layer below pi |
| `03-pi-agent-core.md` | The `Agent` class wrapped by `AgentSession` |
| `05-pi-tui-terminal-ui.md` | TUI primitives used by interactive mode |
| `07-extension-system.md` | Extension API in depth |
| `08-sessions-and-persistence.md` | Session JSONL deep dive |
| `09-version-history.md` | Full changelog roll-up |
| `10-fact-check-report.md` | Cross-doc verification |

### Source map (selected)

| Concern | Path |
|---------|------|
| Binary entry | `src/cli.ts` |
| CLI parsing & help | `src/cli/args.ts` |
| Startup orchestration | `src/main.ts` |
| Constants / dirs / env vars | `src/config.ts` |
| Public SDK surface | `src/index.ts` |
| AgentSession | `src/core/agent-session.ts` |
| AgentSessionRuntime | `src/core/agent-session-runtime.ts` |
| Services factory | `src/core/agent-session-services.ts` |
| Auth | `src/core/auth-storage.ts` |
| Models | `src/core/model-registry.ts`, `src/core/model-resolver.ts` |
| Sessions | `src/core/session-manager.ts`, `src/core/session-cwd.ts` |
| Settings | `src/core/settings-manager.ts` |
| Resources | `src/core/resource-loader.ts`, `src/core/skills.ts`, `src/core/prompt-templates.ts` |
| Extensions | `src/core/extensions/` |
| Tools | `src/core/tools/{index,bash,edit,read,write,grep,find,ls}.ts` |
| Compaction | `src/core/compaction/` |
| Telemetry | `src/core/telemetry.ts` |
| Slash commands | `src/core/slash-commands.ts` |
| HTML export | `src/core/export-html/` |
| Package manager | `src/core/package-manager.ts` |
| Interactive mode | `src/modes/interactive/` |
| Print mode | `src/modes/print/` |
| RPC mode | `src/modes/rpc/` |
