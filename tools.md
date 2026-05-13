# Built-in Tools

Pi ships with **7 built-in tools** in v0.74.0. Four are active by default (`read`, `bash`, `edit`, `write`); three are available but inactive (`grep`, `find`, `ls`). The default set is the minimal primitive set that lets the LLM do anything via bash; the read-only `grep`/`find`/`ls` tools exist for environments where invoking bash is undesirable or for extensions that prefer structured discovery output.

Source: `packages/coding-agent/src/core/tools/{read,write,edit,bash,grep,find,ls}.ts`.

---

## The 4 Default Tools

### `read`

Read the contents of a file. Returns text with line numbers; supports text and binary detection; auto-handles encoding.

```
Parameters:
  path: string    — File path (absolute or relative to cwd)
  offset?: number — Optional starting line (1-indexed)
  limit?: number  — Optional max number of lines

Returns:
  File contents as text with line numbers, or image content for image files
```

Image files are returned as `image` content blocks so vision-capable models can see them.

---

### `write`

Write content to a file, creating or overwriting it entirely. Creates parent directories automatically. Use `edit` for surgical changes.

```
Parameters:
  path: string     — File path
  content: string  — Complete file content
```

---

### `edit`

Make surgical edits via search-and-replace. As of v0.63.2 only the `edits[]` array schema is accepted; sessions created before v0.63.2 are transparently migrated by `prepareArguments`.

```json
{
  "path": "/src/file.ts",
  "edits": [
    { "oldText": "foo", "newText": "bar" },
    { "oldText": "baz", "newText": "qux" }
  ]
}
```

Each `oldText` must appear exactly once in the file. Include surrounding context to make matches unique.

---

### `bash`

Execute a shell command in a persistent process. Working directory and environment persist across calls within a session.

```
Parameters:
  command: string        — Shell command to run
  description: string    — Human-readable label for display
  timeout?: number       — Override default timeout (ms)

Returns:
  { stdout, stderr, exitCode }
```

Key behaviors:
- Full filesystem and command access ("YOLO by default"); no permission popups
- stderr and stdout captured separately; exit code reported
- When output exceeds 2000 lines, full output is persisted to a temp file (v0.65.1) and the LLM receives a truncated view

---

## The 3 Available-but-Inactive Tools

These are read-only file-system tools loaded but not advertised to the model unless enabled (via `--tools` or `settings.json` `tools`):

### `grep`

ripgrep-backed text search. Returns matching files/lines with context. Honors `.gitignore`.

```
Parameters:
  pattern: string
  path?: string
  glob?: string
  caseInsensitive?: boolean
  contextLines?: number
```

### `find`

File-name search by glob. Honors `.gitignore`.

```
Parameters:
  pattern: string    — Glob (e.g. "**/*.ts")
  path?: string
```

### `ls`

Directory listing with file metadata.

```
Parameters:
  path?: string
  showHidden?: boolean
```

---

## Activating Tools

Three knobs:

| Flag / setting | Effect |
|----------------|--------|
| (default) | `read`, `bash`, `edit`, `write` active |
| `--tools read,grep,ls` | Explicit allowlist (built-in, extension, or custom names) |
| `--no-builtin-tools` | Disable built-in tools but keep extension/custom tools |
| `--no-tools` | Disable all tools (built-in and extension) |
| `settings.json` `tools: [...]` | Persistent allowlist |

Example: enable read-only browsing only.

```bash
pi --tools read,grep,find,ls -p "Inventory the auth module"
```

---

## Why So Few Tools?

Standard MCP-style toolsets add 20–50+ tools to every LLM call, consuming 7–9 % of the context window. Pi's philosophy: bash is a universal primitive.

- Browse the web? `curl` or a CLI browser.
- Run sub-agents? `bash` into another `pi` process.
- Bulk file ops? `find`, `xargs`, Python scripts via `bash`.

The minimal set is enough to do everything; tradeoff is that the LLM must know bash.

---

## Tool Result Content

Tool results can mix text and images:

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

## Custom Tools via Extensions

Extensions can add tools that join the same set seen by the model:

```typescript
import { defineTool } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox";  // since v0.69.0

export default function init(pi) {
  pi.registerTool(defineTool({
    name: "run_tests",
    label: "Run Tests",
    description: "Run the test suite and return results",
    parameters: Type.Object({
      filter: Type.Optional(Type.String({ description: "Test filter" }))
    }),
    execute: async (toolCallId, params, signal, onUpdate) => {
      const cmd = params.filter ? `npm test -- ${params.filter}` : "npm test";
      onUpdate?.({ content: [{ type: "text", text: "Running..." }], details: { status: "running" } });
      const result = await runCommand(cmd, { signal });
      return {
        content: [{ type: "text", text: result.output }],
        details: { exitCode: result.exitCode }
      };
    }
  }));
}
```

### Dynamic Registration

```typescript
const unregister = pi.registerTool({ ... });
// Later:
unregister();
```

### Tool Metadata

Add `promptSnippet` and `promptGuidelines` to customize how the tool is described in the system prompt:

```typescript
pi.registerTool({
  name: "my_tool",
  description: "...",
  promptSnippet: "Use this for X when Y.",
  promptGuidelines: "Always call with the full path.",
  // ...
});
```

---

## Execution Hooks

### `before_tool_call`

Intercept and optionally block tool execution:

```typescript
pi.on("before_tool_call", async ({ toolName, args, context }) => {
  if (toolName === "bash" && args.command.includes("rm -rf /")) {
    return { block: true, reason: "Dangerous command blocked" };
  }
  // Return undefined to allow
});
```

### `after_tool_call`

Mutate the result before it is appended to the session and shown to the LLM:

```typescript
pi.on("after_tool_call", async ({ toolName, result, isError }) => {
  if (toolName === "read" && result.content.length > 10000) {
    return {
      content: [{
        type: "text",
        text: result.content[0].text.slice(0, 10000) + "\n...[truncated]"
      }]
    };
  }
  // Return undefined to keep original
});
```

---

## Bash Spawn Hook

Intercept and modify bash commands before execution:

```typescript
pi.setBashSpawnHook(async (command) => {
  await appendToLog(`bash: ${command}`);
  if (command.startsWith("npm install")) {
    return command.replace("npm install", "npm ci --prefer-offline");
  }
  return command;
});
```

---

## Source-of-truth

| File | What it defines |
|------|-----------------|
| `packages/coding-agent/src/core/tools/index.ts` | Tool registry assembly |
| `packages/coding-agent/src/core/tools/{read,write,edit,bash}.ts` | Default tools |
| `packages/coding-agent/src/core/tools/{grep,find,ls}.ts` | Available read-only tools |
| `packages/coding-agent/docs/usage.md` | First-party user reference |
