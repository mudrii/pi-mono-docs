# Extension System

Pi's extension system enables deep customization of the agent through TypeScript modules that can register tools, commands, keyboard shortcuts, event hooks, UI components, and model providers.

Package: `@mariozechner/pi-coding-agent` v0.65.2

## Architecture

### Factory Pattern

Every extension is a TypeScript module whose default export is a **factory function**. The factory receives an `ExtensionAPI` object (`pi`) and uses it to register capabilities:

```ts
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  pi.on("session_start", async (_event, ctx) => {
    ctx.ui.notify("Extension loaded!", "info");
  });
}
```

The factory can be synchronous or async (`Promise<void>`). During factory execution, **action methods** like `pi.sendMessage()` are not yet available -- they throw with "Extension runtime not initialized." Only registration methods (`pi.on()`, `pi.registerTool()`, `pi.registerCommand()`, etc.) and `pi.registerProvider()` are valid at load time.

The type signature:

```ts
type ExtensionFactory = (pi: ExtensionAPI) => void | Promise<void>;
```

### jiti Loading

Extensions are loaded via `@mariozechner/jiti`, a forked version of jiti that supports `virtualModules` for compiled Bun binaries. This allows extensions to be written in TypeScript and loaded at runtime without a build step.

In development (Node.js), jiti uses **aliases** to resolve workspace packages to their filesystem locations. In compiled Bun binary mode, **virtualModules** provides pre-bundled packages directly:

| Package | Available via virtualModules |
|---------|------------------------------|
| `@sinclair/typebox` | Schema definitions for tool parameters |
| `@mariozechner/pi-agent-core` | Core agent types (`AgentMessage`, etc.) |
| `@mariozechner/pi-tui` | TUI components (`Component`, `EditorComponent`, `KeyId`) |
| `@mariozechner/pi-ai` | AI provider types (`Model`, `Api`, `Type`) |
| `@mariozechner/pi-ai/oauth` | OAuth provider registration |
| `@mariozechner/pi-coding-agent` | The coding agent itself (extension types, helpers) |

Each extension is loaded with `moduleCache: false` so that `/reload` picks up changes without restarting.

### Runtime Lifecycle

1. **Load phase** (`loadExtensions()` / `discoverAndLoadExtensions()`):
   - Creates `ExtensionRuntime` with throwing stubs for action methods
   - For each path, creates an `Extension` object (empty maps for handlers, tools, commands, etc.)
   - Creates `ExtensionAPI` wired to the extension object and shared runtime
   - Calls the factory function, which populates the extension's maps
   - Provider registrations are queued in `pendingProviderRegistrations`

2. **Bind phase** (`ExtensionRunner.bindCore()`):
   - Replaces throwing stubs with real action implementations
   - Flushes queued provider registrations to `ModelRegistry`
   - Switches `registerProvider`/`unregisterProvider` to immediate mode

3. **UI Bind phase** (`ExtensionRunner.setUIContext()`):
   - Provides the UI context so `ctx.ui.*` methods work
   - In non-interactive modes (print, RPC), a no-op UI context is used

4. **Command Bind phase** (`ExtensionRunner.bindCommandContext()`):
   - Provides `ExtensionCommandContext` actions (session control, fork, navigate)
   - Only needed in interactive mode

## Discovery Locations and Precedence

Extensions are discovered in this order (first registration wins for conflicts):

1. **Project-local**: `<cwd>/.pi/extensions/` -- auto-discovered
2. **Global**: `~/.pi/agent/extensions/` -- auto-discovered
3. **Configured paths**: from `settings.json` (`extensions` array) or CLI `-e` flag
4. **Package sources**: from `settings.json` (`packages` array) -- npm/git packages with `pi.extensions` in `package.json`

### Discovery Rules within a Directory

Within each extensions directory:

- **Direct files**: `*.ts` or `*.js` files are loaded directly
- **Subdirectory with index**: `subdir/index.ts` or `subdir/index.js`
- **Subdirectory with package.json**: reads the `pi.extensions` field from `package.json` for explicit entry points

No recursion beyond one level. Complex packages must use a `package.json` manifest.

### Deduplication

Resolved absolute paths are tracked. If two discovery locations point to the same file, it is loaded only once.

## ExtensionAPI Methods

The `ExtensionAPI` (`pi`) object provides all registration and action methods.

### Event Subscription: `pi.on(event, handler)`

Subscribe to lifecycle events. Handlers receive the event and an `ExtensionContext`. See the Events section below for the complete list.

### Tool Registration: `pi.registerTool(definition)`

Register a tool callable by the LLM:

```ts
pi.registerTool({
  name: "hello",
  label: "Hello",
  description: "A simple greeting tool",
  promptSnippet: "hello - greets a user by name",
  promptGuidelines: ["Use hello when the user asks to be greeted"],
  parameters: Type.Object({
    name: Type.String({ description: "Name to greet" }),
  }),
  async execute(toolCallId, params, signal, onUpdate, ctx) {
    return {
      content: [{ type: "text", text: `Hello, ${params.name}!` }],
      details: { greeted: params.name },
    };
  },
  renderCall(args, theme) { /* optional custom TUI rendering */ },
  renderResult(result, options, theme) { /* optional custom TUI rendering */ },
});
```

Key fields:
- `name` -- tool name for LLM tool calls. Can match a built-in name to override it
- `promptSnippet` -- one-liner added to the "Available tools" section of the default system prompt
- `promptGuidelines` -- bullet points added to the "Guidelines" section
- `parameters` -- TypeBox schema for parameter validation
- `execute` -- receives the `ExtensionContext` as the last argument
- `renderCall` / `renderResult` -- optional custom TUI renderers; if omitted, built-in renderers are used
- `prepareArguments` -- optional hook to normalize raw model arguments before schema validation (v0.64.0+)

### ToolDefinition.prepareArguments (v0.64.0+)

Attach to any tool definition to normalize raw model arguments before schema validation. Use it to provide backwards compatibility for sessions saved under an old tool schema:

```typescript
pi.registerTool({
  name: "my-tool",
  description: "...",
  parameters: MySchema,
  prepareArguments: (rawArgs) => {
    if (rawArgs.legacyField) {
      return { currentField: rawArgs.legacyField };
    }
    return rawArgs;
  },
  execute: async (toolCallId, params, signal, onUpdate, ctx) => { ... },
});
```

Post-mutation inputs are not re-validated, so ensure your migration always produces valid params.

### Command Registration: `pi.registerCommand(name, options)`

Register a slash command:

```ts
pi.registerCommand("tools", {
  description: "Enable/disable tools",
  getArgumentCompletions(prefix) { return null; },
  async handler(args, ctx) {
    // ctx is ExtensionCommandContext (full session control)
    await ctx.ui.select("Pick one", ["a", "b"]);
  },
});
```

Command handlers receive `ExtensionCommandContext` which includes session control methods not available in regular event handlers.

### Keyboard Shortcut Registration: `pi.registerShortcut(key, options)`

```ts
pi.registerShortcut("ctrl+g", {
  description: "Git status",
  async handler(ctx) {
    const result = await pi.exec("git", ["status"]);
    ctx.ui.notify(result.stdout, "info");
  },
});
```

Reserved key actions (interrupt, clear, exit, submit, etc.) cannot be overridden. Non-reserved built-in bindings can be overridden with a warning.

### CLI Flag Registration: `pi.registerFlag(name, options)`

```ts
pi.registerFlag("verbose", {
  description: "Enable verbose output",
  type: "boolean",
  default: false,
});

// Later:
const verbose = pi.getFlag("verbose"); // boolean | string | undefined
```

### Message Rendering: `pi.registerMessageRenderer(customType, renderer)`

Register custom rendering for `CustomMessageEntry` types:

```ts
pi.registerMessageRenderer("my-status", (message, options, theme) => {
  return { render(width) { return [theme.bold(message.content)]; }, invalidate() {} };
});
```

### Provider Registration: `pi.registerProvider(name, config)`

Register or override a model provider:

```ts
pi.registerProvider("my-proxy", {
  baseUrl: "https://proxy.example.com",
  apiKey: "PROXY_API_KEY",
  api: "anthropic-messages",
  models: [
    {
      id: "claude-sonnet-4-20250514",
      name: "Claude 4 Sonnet (proxy)",
      reasoning: false,
      input: ["text", "image"],
      cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
      contextWindow: 200000,
      maxTokens: 16384,
    },
  ],
});
```

Options:
- **Models provided**: replaces all existing models for this provider
- **Only baseUrl**: overrides URL for existing built-in models
- **OAuth**: registers an OAuth flow for `/login` support
- **streamSimple**: registers a custom API stream handler

During initial load, registrations are queued and applied when `bindCore()` is called. After that, calls take effect immediately without requiring `/reload`.

### Provider Unregistration: `pi.unregisterProvider(name)`

Removes a previously registered provider and restores any built-in models it overrode.

### Action Methods

These are only available after the bind phase (not during factory execution):

| Method | Description |
|--------|-------------|
| `pi.sendMessage(message, options?)` | Send a custom message to the session |
| `pi.sendUserMessage(content, options?)` | Send a user message (always triggers a turn) |
| `pi.appendEntry(customType, data?)` | Append a custom entry for state persistence (not sent to LLM) |
| `pi.setSessionName(name)` | Set session display name |
| `pi.getSessionName()` | Get current session name |
| `pi.setLabel(entryId, label)` | Set/clear a label on an entry |
| `pi.exec(command, args, options?)` | Execute a shell command |
| `pi.getActiveTools()` | Get active tool names |
| `pi.getAllTools()` | Get all configured tools with metadata |
| `pi.setActiveTools(toolNames)` | Set active tools by name |
| `pi.getCommands()` | Get available slash commands |
| `pi.setModel(model)` | Set the current model |
| `pi.getThinkingLevel()` | Get current thinking level |
| `pi.setThinkingLevel(level)` | Set thinking level |
| `pi.events` | Shared `EventBus` for inter-extension communication |

### EventBus

The shared `EventBus` (`pi.events`) allows extensions to communicate with each other without coupling:

```ts
// Extension A
pi.events.emit("my-channel", { action: "refresh" });

// Extension B
const unsub = pi.events.on("my-channel", (data) => {
  console.log("Received:", data);
});
```

## ExtensionContext vs ExtensionCommandContext

### ExtensionContext (read-only)

Passed to all event handlers via the second argument. Provides read access to session state and limited control:

| Property/Method | Description |
|-----------------|-------------|
| `ctx.ui` | UI methods (select, confirm, input, notify, editor, widgets, etc.) |
| `ctx.hasUI` | `false` in print/RPC mode |
| `ctx.cwd` | Current working directory |
| `ctx.sessionManager` | Read-only session manager |
| `ctx.modelRegistry` | Model registry for API key resolution |
| `ctx.model` | Current model (may be undefined) |
| `ctx.signal` | Current agent `AbortSignal`, or `undefined` when no agent turn is active (v0.63.2+) |
| `ctx.isIdle()` | Whether agent is idle |
| `ctx.abort()` | Abort current agent operation |
| `ctx.hasPendingMessages()` | Whether messages are queued |
| `ctx.shutdown()` | Gracefully exit pi |
| `ctx.getContextUsage()` | Get token usage stats |
| `ctx.compact(options?)` | Trigger compaction (fire-and-forget) |
| `ctx.getSystemPrompt()` | Get current effective system prompt |

### ctx.signal (v0.63.2+)

The current agent `AbortSignal`, or `undefined` when no agent turn is active. Forward it into nested model calls, `fetch()`, and other abort-aware async work so that pressing Escape cancels in-flight operations:

```typescript
pi.on("tool_result", async (event, ctx) => {
  if (!ctx.signal) return;
  const data = await fetch("https://api.example.com", { signal: ctx.signal });
  // ...
});
```

### ExtensionCommandContext (full control)

Extends `ExtensionContext` with session control methods. Only available in command handlers:

| Method | Description |
|--------|-------------|
| `ctx.waitForIdle()` | Wait for agent to finish streaming |
| `ctx.newSession(options?)` | Start a new session |
| `ctx.fork(entryId)` | Fork from a specific entry |
| `ctx.navigateTree(targetId, options?)` | Navigate to a different tree point |
| `ctx.switchSession(sessionPath)` | Switch to a different session file |
| `ctx.reload()` | Reload extensions, skills, prompts, themes |

## Event Hooks

Events are organized into categories. Handlers can be async and may return results to modify behavior.

### Resource Events

| Event | When | Return |
|-------|------|--------|
| `resources_discover` | After session_start, allows extensions to provide additional resource paths | `{ skillPaths?, promptPaths?, themePaths? }` |

### Session Events

> **Removed in v0.65.0:** `session_directory`, `session_switch`, and `session_fork` have been removed. See [session_start (unified event)](#session_start-v0650-unified-event) below for migration guidance. `session_directory` has also been removed from the settings API — use `ctx.sessionManager.getSessionFile()` with `path.dirname()` to derive the session's base directory.

| Event | When | Return |
|-------|------|--------|
| `session_start` | On session start, new, resume, fork, or reload | -- |
| `session_before_switch` | Before switching sessions (can cancel) | `{ cancel? }` |
| `session_before_fork` | Before forking (can cancel) | `{ cancel?, skipConversationRestore? }` |
| `session_before_compact` | Before compaction (can cancel or provide custom compaction) | `{ cancel?, compaction? }` |
| `session_compact` | After compaction | -- |
| `session_before_tree` | Before tree navigation (can cancel or provide custom summary) | `{ cancel?, summary?, customInstructions?, label? }` |
| `session_tree` | After tree navigation | -- |
| `session_shutdown` | On process exit | -- |

#### session_start (v0.65.0+ unified event)

`session_switch` and `session_fork` events were removed in v0.65.0. All session transitions now fire `session_start` with a `reason` field:

```typescript
pi.on("session_start", async (event, ctx) => {
  // event.reason: "startup" | "reload" | "new" | "resume" | "fork"
  // event.previousSessionFile: set for "new", "resume", "fork"

  if (event.reason === "fork") {
    ctx.ui.notify(`Forked from ${event.previousSessionFile}`, "info");
  }
});
```

**Migration from removed events:**

```typescript
// Before (removed in v0.65.0)
pi.on("session_switch", async (event, ctx) => { ... });
pi.on("session_fork", async (_event, ctx) => { ... });

// After
pi.on("session_start", async (event, ctx) => {
  if (event.reason === "new" || event.reason === "resume" || event.reason === "fork") {
    // Handle what was previously session_switch or session_fork
  }
});
```

### Agent Events

| Event | When | Return |
|-------|------|--------|
| `context` | Before each LLM call; can modify messages | `{ messages? }` |
| `before_provider_request` | Before provider request is sent; can replace payload | The replacement payload |
| `before_agent_start` | After user submits but before agent loop | `{ message?, systemPrompt? }` |
| `agent_start` | When agent loop starts | -- |
| `agent_end` | When agent loop ends | -- |
| `turn_start` | Start of each turn | -- |
| `turn_end` | End of each turn | -- |
| `message_start` | When a message starts | -- |
| `message_update` | During streaming (token updates) | -- |
| `message_end` | When a message ends | -- |

### Tool Execution Events

| Event | When | Return |
|-------|------|--------|
| `tool_execution_start` | Tool begins executing | -- |
| `tool_execution_update` | Tool streams partial output | -- |
| `tool_execution_end` | Tool finishes executing | -- |

### Tool Interception Events

| Event | When | Return |
|-------|------|--------|
| `tool_call` | Before a tool executes; can block | `{ block?, reason? }` |
| `tool_result` | After a tool executes; can modify result | `{ content?, details?, isError? }` |

Tool call events are typed per built-in tool (`BashToolCallEvent`, `ReadToolCallEvent`, etc.) with type guards:

```ts
import { isToolCallEventType } from "@mariozechner/pi-coding-agent";

pi.on("tool_call", async (event, ctx) => {
  if (isToolCallEventType("bash", event)) {
    // event.input.command is typed as string
    if (event.input.command.includes("rm -rf /")) {
      return { block: true, reason: "Dangerous command blocked" };
    }
  }
});
```

### Input Events

| Event | When | Return |
|-------|------|--------|
| `input` | When user input is received, before processing | `{ action: "continue" }` or `{ action: "transform", text, images? }` or `{ action: "handled" }` |

Transforms chain across extensions. `"handled"` short-circuits (no further processing).

### Model Events

| Event | When | Return |
|-------|------|--------|
| `model_select` | When model changes | -- |

### User Bash Events

| Event | When | Return |
|-------|------|--------|
| `user_bash` | User runs `!command` or `!!command` | `{ operations?, result? }` |

## UI Integration

The `ExtensionUIContext` (`ctx.ui`) provides interactive UI primitives:

### Dialog Methods

```ts
const choice = await ctx.ui.select("Pick one", ["a", "b", "c"], { timeout: 10000 });
const confirmed = await ctx.ui.confirm("Delete?", "This cannot be undone");
const text = await ctx.ui.input("Enter name", "placeholder");
ctx.ui.notify("Something happened", "warning");
const edited = await ctx.ui.editor("Edit config", "initial content");
```

Dialog options support `signal` (AbortSignal) and `timeout` (auto-dismiss with countdown).

### Widgets

```ts
// Simple text widget
ctx.ui.setWidget("my-status", ["Line 1", "Line 2"], { placement: "aboveEditor" });

// Component factory widget
ctx.ui.setWidget("my-widget", (tui, theme) => {
  return { render(w) { return ["custom component"]; }, invalidate() {} };
});

// Clear widget
ctx.ui.setWidget("my-status", undefined);
```

### Status Bar

```ts
ctx.ui.setStatus("my-ext", "Processing...");
ctx.ui.setStatus("my-ext", undefined); // clear
```

### ctx.ui.setHiddenThinkingLabel(label) (v0.64.0+)

Customize the label shown for collapsed thinking blocks in interactive mode:

```typescript
pi.on("session_start", async (_event, ctx) => {
  ctx.ui.setHiddenThinkingLabel("🧠 Reasoning (hidden)");
});
```

No-op in RPC mode. See `examples/extensions/hidden-thinking-label.ts` for a runnable example.

### Custom Components

For full-screen overlays with keyboard focus:

```ts
const result = await ctx.ui.custom((tui, theme, keybindings, done) => {
  return {
    render(width) { return ["My overlay"]; },
    invalidate() {},
    handleInput(data) {
      if (data === "q") done("quit");
    },
  };
}, { overlay: true });
```

### Custom Editor

Replace the input editor:

```ts
import { CustomEditor } from "@mariozechner/pi-coding-agent";

class VimEditor extends CustomEditor {
  handleInput(data: string) {
    if (this.mode === "normal" && data === "i") {
      this.mode = "insert";
      return;
    }
    super.handleInput(data); // App keybindings + text editing
  }
}

ctx.ui.setEditorComponent((tui, theme, keybindings) =>
  new VimEditor(tui, theme, keybindings)
);
```

### Custom Footer, Header, and Theme

```ts
ctx.ui.setFooter((tui, theme, footerData) => {
  return { render(w) { return ["custom footer"]; }, invalidate() {} };
});
ctx.ui.setHeader((tui, theme) => { /* ... */ });
ctx.ui.setTheme("dark");
ctx.ui.setTitle("My Session");
```

### Terminal Input

```ts
const unsub = ctx.ui.onTerminalInput((data) => {
  if (data === "\x1b[A") return { consume: true }; // consume up arrow
  return undefined; // pass through
});
```

## Extension Development Workflow

### Quick Start

1. Create a file (e.g., `~/.pi/agent/extensions/my-ext.ts`)
2. Export a default factory function
3. Start pi -- extensions are auto-discovered from `~/.pi/agent/extensions/`
4. Edit and run `/reload` to pick up changes (no restart needed)

### Using `-e` Flag

```bash
pi -e ./my-extension.ts
pi -e ./extensions-dir/
```

### Multi-file Extensions

For extensions with dependencies, use a subdirectory with `package.json`:

```json
{
  "pi": {
    "extensions": ["./index.ts"]
  }
}
```

Or simply provide an `index.ts` in the subdirectory.

### Extensions with npm Dependencies

Extensions can use npm packages. The `with-deps` example pattern:

1. Create a directory with `package.json` and `pi.extensions` field
2. Run `npm install` in that directory
3. Point pi at the directory via `-e` or settings

## Example Extensions from the Repository

The project includes a rich set of examples in `packages/coding-agent/examples/extensions/`:

| Extension | What it demonstrates |
|-----------|---------------------|
| `hello.ts` | Minimal custom tool registration |
| `tools.ts` | Interactive tool selector with state persistence via `appendEntry` |
| `tool-override.ts` | Overriding the built-in `read` tool with access logging |
| `commands.ts` | Custom slash commands |
| `confirm-destructive.ts` | Blocking destructive bash commands via `tool_call` hook |
| `custom-compaction.ts` | Custom compaction logic via `session_before_compact` |
| `custom-footer.ts` | Custom footer component |
| `custom-provider-*.ts` | Custom provider registration with OAuth |
| `dynamic-tools.ts` | Dynamically enabling/disabling tools |
| `event-bus.ts` | Inter-extension communication via EventBus |
| `input-transform.ts` | Transforming user input via `input` event |
| `modal-editor.ts` | Vim-style editor replacement |
| `plan-mode/` | Multi-file extension with custom workflow |
| `subagent/` | Sub-agent orchestration pattern |
| `status-line.ts` | Status bar widget |
| `bookmark.ts` | Session bookmarking with labels |

Pi-mono itself uses extensions in `.pi/extensions/` for development:

| Extension | Purpose |
|-----------|---------|
| `tps.ts` | Tokens-per-second monitoring |
| `diff.ts` | Diff display utilities |
| `files.ts` | File tracking |
| `redraws.ts` | Redraw counter for TUI debugging |
| `prompt-url-widget.ts` | Prompt URL display widget |

## Error Handling in Extensions

### Load Errors

If an extension fails to load (syntax error, missing module, invalid factory), the error is collected in `LoadExtensionsResult.errors` and reported to the user. Other extensions continue loading.

### Runtime Errors

Handler errors are caught per-handler and reported via `ExtensionRunner.onError()`:

```ts
interface ExtensionError {
  extensionPath: string;
  event: string;
  error: string;
  stack?: string;
}
```

Errors in one extension's handler do not prevent other extensions' handlers from running. The error is emitted to listeners (typically displayed in the UI) but does not crash the session.

### Shortcut and Command Conflicts

- Shortcuts conflicting with reserved built-in actions (interrupt, clear, exit, submit, etc.) are silently skipped with a diagnostic warning
- Shortcuts conflicting with non-reserved built-in actions override the built-in with a warning
- Commands conflicting with built-in slash commands are skipped
- Extension-to-extension conflicts: later-loaded extension wins (last registration)

### Provider Registration Errors

Provider registration validates required fields. Missing `baseUrl` when defining models, or missing `api` type, throws synchronous errors during registration. The extension's factory call will fail, and the error is collected.
