# @mariozechner/pi-agent-core

**Version:** 0.66.1 · **License:** MIT · [npm](https://www.npmjs.com/package/@mariozechner/pi-agent-core)

Stateful agent runtime with tool execution, event streaming, and message queue management. Built on top of `@mariozechner/pi-ai`.

---

## Installation

```bash
npm install @mariozechner/pi-agent-core @mariozechner/pi-ai @sinclair/typebox
```

---

## Quick Start

```typescript
import { Agent } from "@mariozechner/pi-agent-core";
import { getModel } from "@mariozechner/pi-ai";
import { Type } from "@sinclair/typebox";

const model = getModel("anthropic", "claude-opus-4-6");

const agent = new Agent({
  initialState: {
    model,
    systemPrompt: "You are a helpful coding assistant.",
    tools: [
      {
        name: "read_file",
        label: "Read File",
        description: "Read a file's contents",
        parameters: Type.Object({
          path: Type.String()
        }),
        execute: async (id, params, signal) => {
          const content = await fs.readFile(params.path, "utf-8");
          return {
            content: [{ type: "text", text: content }],
            details: { size: content.length }
          };
        }
      }
    ]
  }
});

agent.subscribe(async (event, signal) => {
  if (event.type === "message_update") {
    const e = event.assistantMessageEvent;
    if (e.type === "text_delta") process.stdout.write(e.delta);
  }
});

await agent.prompt("Read the file config.json and summarize it");
await agent.waitForIdle();
```

---

## Agent Class

```typescript
class Agent {
  constructor(opts: AgentOptions);

  // Prompting
  async prompt(message: AgentMessage | string, images?: ImageContent[]): Promise<void>;
  async continue(): Promise<void>;

  // State access
  get state(): AgentState;

  // Direct property access (v0.65.0+)
  toolExecution: "sequential" | "parallel";
  beforeToolCall?: BeforeToolCallFn;
  afterToolCall?: AfterToolCallFn;
  transport?: "sse" | "websocket" | "auto";
  steeringMode: "all" | "one-at-a-time";
  followUpMode: "all" | "one-at-a-time";

  // Active abort signal for the current turn (v0.63.2+)
  get signal(): AbortSignal | undefined;

  // Steering & follow-up
  steer(m: AgentMessage): void;
  followUp(m: AgentMessage): void;
  clearSteeringQueue(): void;
  clearFollowUpQueue(): void;
  clearAllQueues(): void;

  // Control
  abort(): void;
  waitForIdle(): Promise<void>;
  reset(): void;

  // Events
  subscribe(fn: (e: AgentEvent, signal: AbortSignal) => Promise<void> | void): () => void;

  // Session
  get sessionId(): string | undefined;
  set sessionId(v: string | undefined);
  get thinkingBudgets(): ThinkingBudgets | undefined;
  set thinkingBudgets(v: ThinkingBudgets | undefined);
}
```

---

## AgentOptions

```typescript
interface AgentOptions {
  initialState?: Partial<AgentState>;

  // Message conversion: filter custom types to LLM-compatible messages
  convertToLlm?: (messages: AgentMessage[]) => Message[] | Promise<Message[]>;

  // Context transformation (pruning, injection)
  transformContext?: (messages: AgentMessage[], signal?: AbortSignal) => Promise<AgentMessage[]>;

  // Queue modes
  steeringMode?: "all" | "one-at-a-time";     // Default: "one-at-a-time"
  followUpMode?: "all" | "one-at-a-time";      // Default: "one-at-a-time"

  // Tool execution
  toolExecution?: "sequential" | "parallel";   // Default: "parallel"
  beforeToolCall?: BeforeToolCallFn;
  afterToolCall?: AfterToolCallFn;

  // Custom stream (for proxies, local models, etc.)
  streamFn?: StreamFn;

  // Auth
  sessionId?: string;
  getApiKey?: (provider: string) => Promise<string | undefined> | string | undefined;

  // Payload inspection
  onPayload?: (payload: unknown, model: Model<Api>) => unknown | Promise<unknown>;

  // Reasoning
  thinkingBudgets?: ThinkingBudgets;
  transport?: "sse" | "websocket" | "auto";
  maxRetryDelayMs?: number;
}
```

---

## AgentState

```typescript
interface AgentState {
  // Mutable fields
  systemPrompt: string;
  model: Model<any>;
  thinkingLevel: ThinkingLevel;

  // Accessor properties (assignment copies the array)
  tools: AgentTool<any>[];
  messages: AgentMessage[];

  // Readonly (managed by agent loop)
  readonly isStreaming: boolean;
  readonly streamingMessage: AgentMessage | null;  // was streamMessage in ≤0.64
  readonly pendingToolCalls: ReadonlySet<string>;
  readonly errorMessage: string | undefined;        // was error in ≤0.64
}
```

### Migration from ≤0.64

- `streamMessage` → `streamingMessage`
- `error` → `errorMessage`
- `AgentOptions.initialState` no longer accepts readonly fields (`isStreaming`, `streamingMessage`, `pendingToolCalls`, `errorMessage`)

---

## Defining Tools

```typescript
import { Type } from "@sinclair/typebox";

const myTool: AgentTool = {
  name: "search_code",
  label: "Search Code",
  description: "Search for a pattern in source files",
  parameters: Type.Object({
    pattern: Type.String({ description: "Search pattern (regex)" }),
    directory: Type.Optional(Type.String({ description: "Directory to search" }))
  }),

  execute: async (toolCallId, params, signal, onUpdate) => {
    // Optional streaming progress
    onUpdate?.({
      content: [{ type: "text", text: `Searching for: ${params.pattern}...` }],
      details: { status: "running" }
    });

    const results = await searchFiles(params.pattern, params.directory, { signal });

    // Return text result
    return {
      content: [{ type: "text", text: results.join("\n") }],
      details: { count: results.length }
    };
  }
};

// Tool returning an image
const screenshotTool: AgentTool = {
  name: "take_screenshot",
  label: "Take Screenshot",
  description: "Capture the current screen",
  parameters: Type.Object({}),

  execute: async (toolCallId, params, signal) => {
    const imageData = await captureScreen();
    return {
      content: [
        { type: "image", data: imageData, mimeType: "image/png" }
      ],
      details: {}
    };
  }
};

// Tool that throws on error
const riskyTool: AgentTool = {
  name: "delete_file",
  description: "Delete a file",
  parameters: Type.Object({ path: Type.String() }),
  execute: async (id, params, signal) => {
    if (!fs.existsSync(params.path)) {
      throw new Error(`File not found: ${params.path}`);
      // Agent catches this and reports as isError: true
    }
    await fs.promises.unlink(params.path);
    return { content: [{ type: "text", text: "Deleted" }], details: {} };
  }
};
```

---

## Events

```typescript
type AgentEvent =
  | { type: "agent_start" }
  | { type: "agent_end"; messages: AgentMessage[] }
  | { type: "turn_start" }
  | { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
  | { type: "message_start"; message: AgentMessage }
  | { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
  | { type: "message_end"; message: AgentMessage }
  | { type: "tool_execution_start"; toolCallId: string; toolName: string; args: any }
  | { type: "tool_execution_update"; toolCallId: string; toolName: string; args: any; partialResult: any }
  | { type: "tool_execution_end"; toolCallId: string; toolName: string; result: any; isError: boolean }
  | { type: "state_change"; state: AgentState };   // emitted when state fields change

// Subscribe (v0.65.0+: async listener with signal)
const unsubscribe = agent.subscribe(async (event, signal) => {
  if (event.type === "message_update") {
    const e = event.assistantMessageEvent;
    if (e.type === "text_delta") console.log(e.delta);
    if (e.type === "thinking_delta") console.log("[thinking]", e.delta);
    if (e.type === "toolcall_end") console.log("Tool:", e.toolCall.name);
  }
});

// Cleanup
unsubscribe();
```

---

## Steering and Follow-up

```typescript
// Interrupt: queued after the current tool-call batch fully completes (v0.58.4+)
agent.steer({
  role: "user",
  content: "Stop! Focus on the authentication bug instead.",
  timestamp: Date.now()
});

// Follow-up: runs after agent finishes current work
agent.followUp({
  role: "user",
  content: "Also add tests for what you just implemented.",
  timestamp: Date.now()
});

// Control modes (v0.65.0+: direct property access)
agent.steeringMode = "one-at-a-time";   // Process one steering per turn
agent.steeringMode = "all";             // Process all steering at once

agent.clearSteeringQueue();
agent.clearFollowUpQueue();
agent.clearAllQueues();
```

---

## Tool Execution Hooks

```typescript
const agent = new Agent({
  beforeToolCall: async (context, signal) => {
    const { toolCall, args } = context;

    // Block dangerous operations
    if (toolCall.name === "bash" && args.command?.includes("rm -rf /")) {
      return { block: true, reason: "Dangerous command blocked" };
    }

    // Allow (return undefined)
  },

  afterToolCall: async (context, signal) => {
    const { toolCall, result, isError } = context;

    // Truncate large results
    if (result.content[0]?.type === "text" && result.content[0].text.length > 50000) {
      return {
        content: [{
          type: "text",
          text: result.content[0].text.slice(0, 50000) + "\n...[truncated]"
        }]
      };
    }

    // Keep original (return undefined)
  }
});
```

---

## Custom Message Types

Extend agent messages via declaration merging:

```typescript
declare module "@mariozechner/pi-agent-core" {
  interface CustomAgentMessages {
    screenshot: {
      role: "screenshot";
      timestamp: number;
      imageData: string;        // Base64
      description: string;
    };
    notification: {
      role: "notification";
      timestamp: number;
      text: string;
      level: "info" | "warn" | "error";
    };
  }
}

// Now AgentMessage includes these types
const messages: AgentMessage[] = [
  { role: "user", content: "Hello", timestamp: Date.now() },
  { role: "notification", text: "Build succeeded", level: "info", timestamp: Date.now() }
];

// convertToLlm must filter custom types
const agent = new Agent({
  initialState: { messages },
  convertToLlm: (msgs) => msgs.flatMap(m => {
    if (m.role === "notification") return [];  // Filter out
    if (m.role === "screenshot") {
      return [{
        role: "user",
        content: [
          { type: "text", text: m.description },
          { type: "image", data: m.imageData, mimeType: "image/png" }
        ],
        timestamp: m.timestamp
      }];
    }
    return [m];  // Pass through standard messages
  })
});
```

---

## Low-Level Agent Loop

For full control without the `Agent` class:

```typescript
import { agentLoop, runAgentLoop } from "@mariozechner/pi-agent-core";

// Event-based iteration
const stream = agentLoop(
  [{ role: "user", content: "Hello", timestamp: Date.now() }],
  { systemPrompt: "...", messages: [], tools: [] },
  { model, convertToLlm: (msgs) => msgs }
);

for await (const event of stream) {
  if (event.type === "message_update") {
    // Handle streaming
  }
}

const finalMessages = await stream.result();

// Or imperative
await runAgentLoop(
  prompts,
  context,
  config,
  (event) => console.log(event)
);
```

---

## Proxy Support

Route all LLM calls through a proxy server:

```typescript
import { streamProxy } from "@mariozechner/pi-agent-core";

const agent = new Agent({
  streamFn: (model, context, options) =>
    streamProxy(model, context, {
      ...options,
      authToken: "my-bearer-token",
      proxyUrl: "https://proxy.example.com/api/stream"
    })
});
```

The proxy receives the full context, handles the LLM call server-side, and streams events back.

---

## Breaking Changes (v0.65.0)

### Removed Mutator Methods

All setter methods on `Agent` were replaced with direct property and state access:

| Removed | Replacement |
|---------|-------------|
| `agent.setSystemPrompt(v)` | `agent.state.systemPrompt = v` |
| `agent.setModel(m)` | `agent.state.model = m` |
| `agent.setThinkingLevel(l)` | `agent.state.thinkingLevel = l` |
| `agent.setTools(t)` | `agent.state.tools = t` |
| `agent.replaceMessages(msgs)` | `agent.state.messages = msgs` |
| `agent.appendMessage(msg)` | `agent.state.messages.push(msg)` |
| `agent.clearMessages()` | `agent.state.messages = []` |
| `agent.setToolExecution(m)` | `agent.toolExecution = m` |
| `agent.setBeforeToolCall(fn)` | `agent.beforeToolCall = fn` |
| `agent.setAfterToolCall(fn)` | `agent.afterToolCall = fn` |
| `agent.setTransport(t)` | `agent.transport = t` |
| `agent.setSteeringMode(m)` | `agent.steeringMode = m` |
| `agent.getSteeringMode()` | `agent.steeringMode` |
| `agent.setFollowUpMode(m)` | `agent.followUpMode = m` |
| `agent.getFollowUpMode()` | `agent.followUpMode` |

### subscribe() — Async with Signal

```typescript
agent.subscribe(async (event, signal) => { ... });
```

`agent_end` is now the final emitted event. `waitForIdle()`, `prompt()`, and `continue()` settle after awaited listeners complete.

---

## Agent.signal (v0.63.2+)

Active `AbortSignal` for the current turn (`undefined` when idle). Forward into nested async work for cancellation support.

```typescript
const result = await fetch(url, { signal: agent.signal });
```

---

## AgentTool.prepareArguments (v0.64.0+)

Hook to normalize raw model arguments before schema validation. Use for backwards compatibility with old session schemas.

```typescript
const myTool: AgentTool = {
  name: "my_tool",
  parameters: Type.Object({ value: Type.String() }),
  prepareArguments: (args) => {
    // Normalize legacy field names before validation
    const raw = args as any;
    return { value: raw.val ?? raw.value };
  },
  execute: async (id, params, signal) => { ... }
};
```
