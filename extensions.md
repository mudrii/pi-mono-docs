# Extensions

Pi's extension system is the primary way to add functionality. Extensions are TypeScript files (or npm packages) that integrate with the agent lifecycle.

---

## Extension Loading Order

1. Built-in tools (`read`, `write`, `edit`, `bash`)
2. User-global extensions (`~/.pi/agent/extensions/`)
3. Project-level extensions (`.pi/extensions/`)
4. Installed packages (`pi install <package>`)

Extensions are loaded in this order; project extensions can override user-global ones.

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
import type { ExtensionContext } from "@mariozechner/pi-coding-agent";
import { Type } from "@sinclair/typebox";

export default async function init(ctx: ExtensionContext) {
  // Register a tool
  ctx.registerTool({
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

  // Register a slash command
  ctx.registerCommand({
    name: "/fetch",
    description: "Fetch a URL",
    completions: () => [],  // No completions
    execute: async (args) => {
      ctx.appendUserMessage(`Please fetch and summarize: ${args}`);
    }
  });
}
```

---

## ExtensionContext API

### Tool Registration

```typescript
// Register a tool (returns unregister function)
const unregister = ctx.registerTool(tool: AgentTool): () => void;

// Get all registered tools (includes parameters)
ctx.getAllTools(): AgentTool[];
```

### Command Registration

```typescript
ctx.registerCommand({
  name: string,               // "/my-command"
  description: string,
  completions: (partial: string) => string[],
  execute: (args: string) => Promise<void> | void
});
```

### Event Hooks

```typescript
ctx.on(event: string, handler: Function): void;
```

| Event | Description |
|-------|-------------|
| `init` | Extension loaded |
| `session_start` | New session begins |
| `session_end` | Session ends |
| `session_directory` | Customize session storage path |
| `before_tool_call` | Before any tool executes (can block) |
| `after_tool_call` | After any tool executes (can mutate result) |
| `before_provider_request` | Before LLM API call (inspect/replace payload) |
| `terminal_input` | Raw terminal input received |
| `resources_discover` | Supply additional skills, prompts, themes |

### Session Management

```typescript
// Switch to another session
await ctx.switchSession(sessionId: string): Promise<void>;

// Reload all extensions
await ctx.reload(): Promise<void>;

// Context management
await ctx.compact(): Promise<void>;
ctx.getContextUsage(): { used: number; total: number };
```

### Message Queue

```typescript
// Queue a steering message (interrupts current work)
ctx.steer(message: string | AgentMessage): void;

// Queue a follow-up (after current work completes)
ctx.followUp(message: string | AgentMessage): void;

// Programmatically paste into editor
ctx.pasteToEditor(text: string): void;
```

### UI: Custom Overlays

```typescript
// Show a capturing overlay (blocks input to underlying TUI)
ctx.ui.showOverlay({
  component: myComponent,
  position: "center",           // "center" | "top" | "bottom" | percent
  size: { width: "50%", height: "auto" },
  onClose: () => {},
  capturing: true               // Capture all input
});

// Non-capturing overlay (informational, doesn't block input)
ctx.ui.showOverlay({
  component: statusComponent,
  position: "bottom",
  capturing: false
});
```

### Provider Registration

```typescript
// Register a custom LLM provider
ctx.registerProvider({
  id: "my-provider",
  name: "My Provider",
  streamSimple: (model, context, options) => customStream(model, context, options)
});

// Remove a provider
ctx.unregisterProvider("my-provider");
```

### Session Labeling

```typescript
// Set a display label for the current session
ctx.setLabel("Code Review: auth.ts");
```

---

## Lifecycle Events (Detail)

### `before_provider_request`

Inspect or replace the LLM payload before it's sent:

```typescript
ctx.on("before_provider_request", async (payload, model) => {
  // Log the payload
  console.log("Sending to", model.id, JSON.stringify(payload).length, "bytes");

  // Return undefined to keep original, or return modified payload
  return {
    ...payload,
    temperature: 0.2  // Override temperature
  };
});
```

### `terminal_input`

Intercept raw terminal input:

```typescript
ctx.on("terminal_input", (data: Buffer) => {
  // Return true to consume the input (prevent normal handling)
  if (data.toString() === "\x1b[A") {  // Up arrow
    handleCustomUpArrow();
    return true;
  }
  return false;
});
```

### `session_directory`

Customize where session files are stored:

```typescript
ctx.on("session_directory", (defaultPath: string) => {
  // Return custom path, or undefined for default
  return `/my/shared/sessions/${projectName}`;
});
```

### `resources_discover`

Supply additional resources:

```typescript
ctx.on("resources_discover", () => ({
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
    { name: "my-theme", colors: { ... } }
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
    "@mariozechner/pi-coding-agent": ">=0.50.0"
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
ctx.setBashSpawnHook(async (command: string) => {
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
ctx.registerTool({
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
