# Extensions

Pi's extension system is the primary way to add functionality. Extensions are TypeScript files (or npm packages) that integrate with the agent lifecycle. (Updated for v0.66.1.)

---

## Extension Loading Order

1. Built-in tools (`read`, `bash`, `edit`, `write` active by default; `grep`, `find`, `ls` available)
2. Project-level extensions (`.pi/extensions/`) — auto-discovered
3. User-global extensions (`~/.pi/agent/extensions/`) — auto-discovered
4. Configured `extensions` paths from `settings.json` or `--extension`
5. Installed packages (`pi install <source>`) listed in `settings.json` `packages`

First registration wins on conflicts. Project extensions take precedence over user-global; explicit `extensions` paths layered next; package extensions last. Pi loads each extension only once even if multiple discovery routes resolve to the same absolute path.

---

## Installing Extensions

```bash
pi install @scope/package-name         # npm package
pi install github.com/user/repo        # GitHub URL
pi install git@github.com:user/repo    # SSH git URL
pi install ./local/path                # Local directory
pi update                              # Update all
pi list                                # List installed
pi remove @scope/package-name          # Remove
pi config                              # TUI: enable/disable resources
```

---

## Writing an Extension

An extension is a JavaScript/TypeScript file with a default export function:

```typescript
// my-extension.ts
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { defineTool } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox"; // v0.69.0+ (was @sinclair/typebox; legacy alias kept)

export default async function init(pi: ExtensionAPI) {
  // Register a tool using defineTool() for full TypeScript type inference (v0.65.0+)
  const fetchTool = defineTool({
    name: "fetch_url",
    label: "Fetch URL",
    description: "Fetch content from a URL",
    parameters: Type.Object({
      url: Type.String({ description: "URL to fetch" }),
      format: Type.Optional(Type.Union([
        Type.Literal("text"),
        Type.Literal("json")
      ]))
    }),
    execute: async (toolCallId, params, signal) => {
      const response = await fetch(params.url, { signal });
      const content = params.format === "json"
        ? JSON.stringify(await response.json(), null, 2)
        : await response.text();
      return {
        content: [{ type: "text", text: content }],
        details: { status: response.status, url: params.url }
      };
    }
  });
  pi.registerTool(fetchTool);

  // Register a slash command
  pi.registerCommand({
    name: "/fetch",
    description: "Fetch a URL",
    completions: () => [],  // No completions
    execute: async (args, ctx) => {
      ctx.sendMessage(`Please fetch and summarize: ${args}`);
    }
  });
}
```

---

## ExtensionAPI Methods

The default-exported factory receives an `ExtensionAPI` (`pi`). Each event handler receives an `ExtensionContext` (`ctx`) scoped to the current turn/session.

### Tool Registration

```typescript
// Create a standalone tool definition with type inference (v0.65.0+)
const myTool = defineTool({ name: "...", parameters: ..., execute: ... });

// Register a tool (returns unregister function)
const unregister = pi.registerTool(tool: AgentTool): () => void;

// Get all registered tools (includes parameters)
pi.getAllTools(): AgentTool[];
```

#### `prepareArguments` hook (v0.64.0)

Attach a `prepareArguments` hook to normalize or migrate raw model arguments before schema validation:

```typescript
pi.registerTool({
  name: "my_tool",
  parameters: ...,
  prepareArguments: (rawArgs) => {
    // Normalize legacy argument shapes before validation
    if (rawArgs.oldField) {
      return { newField: rawArgs.oldField };
    }
    return rawArgs;
  },
  execute: async (toolCallId, params, signal) => { ... }
});
```

### Command Registration

```typescript
pi.registerCommand({
  name: string,               // "my-command" (slash added automatically)
  description: string,
  completions: (partial: string) => string[],
  execute: (args: string, ctx: ExtensionContext) => Promise<void> | void
});
```

### Event Hooks

```typescript
pi.on(event: string, handler: (event, ctx) => unknown): void;
```

| Event | Description |
|-------|-------------|
| `init` | Extension loaded |
| `session_start` | Session begins (startup, reload, new, resume, or fork) — see below |
| `session_end` | Session ends |
| `before_tool_call` | Before any tool executes (can block) |
| `after_tool_call` | After any tool executes (can mutate result) |
| `before_provider_request` | Before LLM API call (inspect/replace payload) |
| `terminal_input` | Raw terminal input received |
| `resources_discover` | Supply additional skills, prompts, themes |

> **Note (v0.65.0):** `session_switch` and `session_fork` events were removed. Use `session_start` with `event.reason`. `session_directory` was also removed from the extension API.

#### `session_start` event

```typescript
pi.on("session_start", async (event, ctx) => {
  // event.reason: "startup" | "reload" | "new" | "resume" | "fork"
  // event.previousSessionFile: set for "new", "resume", and "fork"
  if (event.reason === "fork") {
    ctx.setLabel(`Fork of ${event.previousSessionFile}`);
  }
});
```

### Cancellation Signal (v0.63.2)

The per-handler `ctx.signal` is an `AbortSignal` tied to the active agent turn. Forward it to nested `fetch()` or model API calls for proper cancellation:

```typescript
pi.on("session_start", async (event, ctx) => {
  const response = await fetch("https://api.example.com/data", { signal: ctx.signal });
});
```

### Session Management (from `ctx` in handlers)

```typescript
// Switch to another session
await ctx.switchSession(sessionId: string): Promise<void>;

// Reload all extensions
await ctx.reload(): Promise<void>;

// Context management
await ctx.compact(): Promise<void>;
ctx.getContextUsage(): { used: number; total: number };
```

### Message Queue (from `ctx`)

```typescript
// Queue a steering message (interrupts current work)
ctx.steer(message: string | AgentMessage): void;

// Queue a follow-up (after current work completes)
ctx.followUp(message: string | AgentMessage): void;

// Programmatically paste into editor
ctx.pasteToEditor(text: string): void;

// Send a user message immediately
ctx.sendMessage(text: string): void;
```

### UI Methods (from `ctx`)

```typescript
// Set a custom label for the collapsed thinking block (v0.64.0)
ctx.ui.setHiddenThinkingLabel("Reasoning...");

// Capturing overlay (blocks input to underlying TUI)
ctx.ui.showOverlay({
  component: myComponent,
  position: "center",
  size: { width: "50%", height: "auto" },
  onClose: () => {},
  capturing: true
});

// Notification (info | warn | error)
ctx.ui.notify("Done!", "info");
```

### Provider Registration (on `pi`, at load time or runtime)

```typescript
pi.registerProvider({
  id: "my-provider",
  name: "My Provider",
  streamSimple: (model, context, options) => customStream(model, context, options)
});

pi.unregisterProvider("my-provider");
```

### Session Labeling (from `ctx`)

```typescript
ctx.setLabel("Code Review: auth.ts");
```

---

## Lifecycle Events (Detail)

### `before_provider_request`

Inspect or replace the LLM payload before it's sent:

```typescript
pi.on("before_provider_request", async (event, ctx) => {
  // Inspect or replace the LLM payload
  return {
    ...event.payload,
    temperature: 0.2
  };
});
```

### `terminal_input`

Intercept raw terminal input:

```typescript
pi.on("terminal_input", (event, ctx) => {
  // Return true to consume the input
  if (event.data.toString() === "\x1b[A") {  // Up arrow
    handleCustomUpArrow();
    return true;
  }
  return false;
});
```

### `resources_discover`

Supply additional resources:

```typescript
pi.on("resources_discover", () => ({
  skills: [
    {
      name: "my-skill",
      description: "Custom skill",
      content: "# My Skill\n..."
    }
  ],
  prompts: [
    { name: "/standup", content: "Generate a standup report..." }
  ],
  themes: [
    { name: "my-theme", colors: { /* ... */ } }
  ]
}));
```

---

## Extension Package Structure

A pip-installable extension package:

```
my-pi-extension/
├── package.json
├── index.ts           # Extension entry point (default export)
├── tsconfig.json
└── README.md
```

```json
// package.json
{
  "name": "@myorg/pi-extension-name",
  "version": "1.0.0",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "peerDependencies": {
    "@earendil-works/pi-coding-agent": ">=0.50.0"
  }
}
```

---

## Examples

The monorepo includes 50+ example extensions in `packages/coding-agent/examples/extensions/`:

| Example | Description |
|---------|-------------|
| `with-deps` | Extension with npm dependencies |
| `custom-provider-anthropic` | Custom Anthropic provider variant |
| `custom-provider-gitlab-duo` | GitLab Duo as a provider |
| `custom-provider-qwen-cli` | Qwen CLI as a provider |
| `titlebar-spinner` | Custom loading spinner in title bar |
| `antigravity-image-gen` | Image generation via Google Antigravity |
| `qwen-cli-oauth` | Qwen CLI OAuth flow |
| And many more... | See `examples/extensions/` directory |

---

## Bash Spawn Hook

Intercept and modify bash commands before they execute:

```typescript
pi.setBashSpawnHook(async (command: string) => {
  // Must return the command to run (can be modified)
  // Log, transform, or replace the command
  if (command.includes("sudo")) {
    throw new Error("sudo is not allowed");
  }
  return command;
});
```

---

## Tool Output Customization

Control how tool output is displayed:

```typescript
pi.registerTool({
  name: "my_tool",
  // ...
  // Show output expanded by default
  defaultExpanded: true,

  // Custom renderer for TUI output
  render: (result, isStreaming) => {
    return formatResultAsTable(result);
  }
});
```

Tool output in the TUI can be expanded/collapsed with `Space`. Use `e` to expand all, `c` to collapse all.
