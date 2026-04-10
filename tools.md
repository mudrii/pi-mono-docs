# Built-in Tools

Pi ships with exactly 4 built-in tools. This is a deliberate design choice: the minimal primitive set that lets the LLM do anything needed via bash. (Updated for v0.66.1.)

---

## The 4 Core Tools

### `read_file`

Read the contents of a file.

```
Parameters:
  path: string    — File path (absolute or relative to cwd)

Returns:
  File contents as text, with line numbers
```

Supports encoding detection. Returns an error message if the file doesn't exist.

---

### `write_file`

Write content to a file, creating it or overwriting it entirely.

```
Parameters:
  path: string     — File path (absolute or relative to cwd)
  content: string  — Complete file content

Returns:
  Success confirmation with path
```

Creates parent directories automatically. Replaces the entire file; use `edit_file` for surgical changes.

---

### `edit_file`

Make surgical edits to an existing file using search-and-replace. As of v0.63.2, only the `edits[]` array schema is accepted. The legacy single-edit `oldText`/`newText` at the top level was removed; sessions created before v0.63.2 are transparently migrated via `prepareArguments`.

```
Parameters:
  path: string         — File path
  edits: Array         — List of edits to apply

  Each edit:
    oldText: string    — Exact text to find (including whitespace)
    newText: string    — Replacement text

Returns:
  Success confirmation, or error if any oldText not found
```

**Current schema (v0.63.2+):**

```json
{
  "path": "/src/file.ts",
  "edits": [
    { "oldText": "foo", "newText": "bar" },
    { "oldText": "baz", "newText": "qux" }
  ]
}
```

Each `oldText` must appear exactly once in the file. Include sufficient surrounding context to ensure uniqueness.

---

### `bash`

Execute a shell command.

```
Parameters:
  command: string        — Shell command to run
  description: string    — Human-readable label for display

Returns:
  {stdout, stderr, exitCode}
```

Runs synchronously in a persistent shell process. The working directory and environment persist between calls within the same session.

**Key behaviors:**
- Full filesystem and command access ("YOLO by default")
- No permission popups or sandboxing
- Timeout enforced (configurable)
- stderr and stdout captured separately
- Exit codes reported
- When output exceeds 2000 lines, the full output is persisted to a temp file (v0.65.1). The LLM receives a truncated view, but the complete output remains accessible.

**Usage patterns:**
```bash
# Installing dependencies
npm install

# Running tests
npm test -- --filter my-test

# Git operations
git diff HEAD
git commit -am "fix: resolve edge case in parser"

# File discovery
find . -name "*.ts" -not -path "*/node_modules/*"
rg "deprecated" --type ts

# Build and lint
npm run build && npm run check
```

---

## Why Only 4 Tools?

Standard MCP servers add 20–50+ tools to every LLM call, consuming 7–9% of the context window. Pi's philosophy: bash is a universal primitive.

Need to browse the web? `curl` or a browser CLI.
Need to run sub-agents? `bash` into another `pi` process.
Need to manage files in bulk? `find`, `xargs`, Python scripts via `bash`.

The 4 tools are enough to do everything; the tradeoff is that the LLM must know how to use bash.

---

## Tool Result Content

Tool results can include both text and images:

```typescript
{
  content: [
    { type: "text", text: "File contents..." },
    { type: "image", data: "base64...", mimeType: "image/png" }
  ],
  isError: false
}
```

Vision-capable models can receive screenshots, diagrams, and file previews as tool results.

---

## Custom Tools (Extensions)

Extensions can add additional tools:

```typescript
import { Type } from "@sinclair/typebox";

// In extension init
ctx.registerTool({
  name: "run_tests",
  label: "Run Tests",
  description: "Run the test suite and return results",
  parameters: Type.Object({
    filter: Type.Optional(Type.String({
      description: "Test filter pattern"
    }))
  }),
  execute: async (toolCallId, params, signal, onUpdate) => {
    const { filter } = params;
    const cmd = filter ? `npm test -- ${filter}` : "npm test";

    // Optional streaming update
    onUpdate?.({
      content: [{ type: "text", text: "Running tests..." }],
      details: { status: "running" }
    });

    const result = await runCommand(cmd, { signal });
    return {
      content: [{ type: "text", text: result.output }],
      details: { exitCode: result.exitCode }
    };
  }
});
```

Tools registered via extensions are automatically included in the system prompt tool list.

### Dynamic Registration (v0.55.4+)

Tools can be registered and unregistered at runtime without reloading:

```typescript
const unregister = ctx.registerTool({ ... });
// Later:
unregister();
```

### Tool Metadata

Add `promptSnippet` and `promptGuidelines` to customize how the tool is described in the system prompt:

```typescript
ctx.registerTool({
  name: "my_tool",
  description: "...",
  promptSnippet: "Use this for X when Y.",
  promptGuidelines: "Always call with the full path.",
  // ...
});
```

---

## Tool Execution Hooks

Extensions can intercept tool execution:

### `beforeToolCall`

Called before any tool executes. Can block execution:

```typescript
ctx.on("before_tool_call", async ({ toolName, args, context }) => {
  if (toolName === "bash" && args.command.includes("rm -rf")) {
    return { block: true, reason: "Dangerous command blocked" };
  }
  // Return undefined to allow
});
```

### `afterToolCall`

Called after tool executes. Can mutate the result:

```typescript
ctx.on("after_tool_call", async ({ toolName, result, isError }) => {
  if (toolName === "read_file" && result.content.length > 10000) {
    return {
      content: [{
        type: "text",
        text: result.content[0].text.slice(0, 10000) + "\n...[truncated]"
      }]
    };
  }
  // Return undefined to keep original result
});
```

---

## Bash Spawn Hook

Intercept and modify bash commands before execution:

```typescript
ctx.setBashSpawnHook(async (command) => {
  // Log all commands
  await appendToLog(`bash: ${command}`);

  // Optionally transform
  if (command.startsWith("npm install")) {
    return command.replace("npm install", "npm ci --prefer-offline");
  }
  return command;
});
```
