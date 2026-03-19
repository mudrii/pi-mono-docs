# pi-agent-core (`@mariozechner/pi-agent-core`)

Package: `@mariozechner/pi-agent-core` v0.60.0
Source: `packages/agent/`
License: MIT | Node >= 20.0.0
Dependency: `@mariozechner/pi-ai ^0.60.0`

Stateful agent with tool execution and event streaming. Built on `@mariozechner/pi-ai`, this package provides the `Agent` class and a low-level `agentLoop()` API for driving multi-turn LLM conversations with tool calls, steering, and follow-up queues.

---

## Package Exports

```
src/index.ts
  ├── agent.ts       → Agent class
  ├── agent-loop.ts  → agentLoop(), agentLoopContinue(), runAgentLoop(), runAgentLoopContinue()
  ├── proxy.ts       → streamProxy(), ProxyAssistantMessageEvent, ProxyStreamOptions
  └── types.ts       → AgentTool, AgentEvent, AgentState, AgentContext, AgentLoopConfig, etc.
```

---

## Agent Class

The `Agent` class wraps the agent loop with state management, event subscription, and queue-based steering/follow-up. It calls `streamSimple` from pi-ai by default (overridable via `streamFn`).

### Constructor Options (`AgentOptions`)

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `initialState` | `Partial<AgentState>` | see below | System prompt, model, tools, messages, thinking level |
| `convertToLlm` | `(msgs) => Message[]` | filters to user/assistant/toolResult | Converts `AgentMessage[]` to LLM-compatible `Message[]` before each call |
| `transformContext` | `(msgs, signal?) => Promise<AgentMessage[]>` | none | Pre-processing hook (pruning, external context injection) applied before `convertToLlm` |
| `streamFn` | `StreamFn` | `streamSimple` | Custom stream function (for proxy backends) |
| `steeringMode` | `"all" \| "one-at-a-time"` | `"one-at-a-time"` | How steering messages are dequeued |
| `followUpMode` | `"all" \| "one-at-a-time"` | `"one-at-a-time"` | How follow-up messages are dequeued |
| `sessionId` | `string` | none | Forwarded to providers for session-based caching |
| `getApiKey` | `(provider) => Promise<string \| undefined>` | none | Dynamic API key resolution per call (for expiring OAuth tokens) |
| `onPayload` | `SimpleStreamOptions["onPayload"]` | none | Inspect or replace provider payloads before sending |
| `thinkingBudgets` | `ThinkingBudgets` | none | Custom token budgets per thinking level |
| `transport` | `Transport` | `"sse"` | Preferred transport (`"sse"`, `"websocket"`, `"auto"`) |
| `maxRetryDelayMs` | `number` | `60000` | Cap on server-requested retry delays; 0 disables |
| `toolExecution` | `ToolExecutionMode` | `"parallel"` | `"parallel"` or `"sequential"` |
| `beforeToolCall` | hook | none | Preflight hook; can block tool execution |
| `afterToolCall` | hook | none | Post-execution hook; can mutate tool results |

### Default State

```typescript
{
  systemPrompt: "",
  model: getModel("google", "gemini-2.5-flash-lite-preview-06-17"),
  thinkingLevel: "off",
  tools: [],
  messages: [],
  isStreaming: false,
  streamMessage: null,
  pendingToolCalls: new Set<string>(),
  error: undefined,
}
```

### AgentState Interface

```typescript
interface AgentState {
  systemPrompt: string;
  model: Model<any>;
  thinkingLevel: ThinkingLevel;           // "off" | "minimal" | "low" | "medium" | "high" | "xhigh"
  tools: AgentTool<any>[];
  messages: AgentMessage[];
  isStreaming: boolean;
  streamMessage: AgentMessage | null;     // Partial message during streaming
  pendingToolCalls: Set<string>;          // Tool call IDs currently executing
  error?: string;
}
```

### Methods

**Prompting:**

| Method | Description |
|--------|-------------|
| `prompt(text, images?)` | Send a text prompt, optionally with images |
| `prompt(message)` | Send an `AgentMessage` directly |
| `prompt(messages)` | Send an array of `AgentMessage` |
| `continue()` | Resume from current context (last message must be user or toolResult) |

Both `prompt()` and `continue()` throw if called while the agent is already streaming. Use `steer()` or `followUp()` to queue messages during streaming.

**State Mutators:**

| Method | Description |
|--------|-------------|
| `setSystemPrompt(v)` | Replace system prompt |
| `setModel(m)` | Switch model |
| `setThinkingLevel(l)` | Set thinking/reasoning level |
| `setTools(t)` | Replace tool set |
| `setToolExecution(v)` | Switch between `"parallel"` and `"sequential"` |
| `setBeforeToolCall(fn)` | Set/clear beforeToolCall hook |
| `setAfterToolCall(fn)` | Set/clear afterToolCall hook |
| `replaceMessages(ms)` | Replace entire message history |
| `appendMessage(m)` | Append a single message |
| `clearMessages()` | Clear message history |
| `reset()` | Clear messages, queues, streaming state, error |

**Steering and Follow-up:**

| Method | Description |
|--------|-------------|
| `steer(m)` | Queue a steering message (delivered after current turn's tool calls finish) |
| `followUp(m)` | Queue a follow-up message (delivered only when agent has no more work) |
| `setSteeringMode(mode)` | `"all"` or `"one-at-a-time"` |
| `setFollowUpMode(mode)` | `"all"` or `"one-at-a-time"` |
| `getSteeringMode()` | Current steering mode |
| `getFollowUpMode()` | Current follow-up mode |
| `clearSteeringQueue()` | Drop queued steering messages |
| `clearFollowUpQueue()` | Drop queued follow-up messages |
| `clearAllQueues()` | Drop all queued messages |
| `hasQueuedMessages()` | True if either queue is non-empty |

**Control:**

| Method | Description |
|--------|-------------|
| `abort()` | Cancel current operation via AbortController |
| `waitForIdle()` | Returns a promise that resolves when the current prompt finishes |

**Events:**

```typescript
const unsubscribe = agent.subscribe((event: AgentEvent) => {
  console.log(event.type);
});
unsubscribe(); // remove listener
```

**Properties:**

| Property | Type | Description |
|----------|------|-------------|
| `state` | `AgentState` (readonly) | Current agent state |
| `sessionId` | `string \| undefined` | Get/set session ID for provider caching |
| `thinkingBudgets` | `ThinkingBudgets \| undefined` | Get/set custom thinking token budgets |
| `transport` | `Transport` | Current transport preference |
| `toolExecution` | `ToolExecutionMode` | Current tool execution mode |
| `streamFn` | `StreamFn` | Stream function (public, mutable) |

---

## Agent Loop Execution Model

The agent loop is a turn-based execution engine. Each turn consists of one LLM call followed by optional tool execution. The loop continues until the LLM produces a response with no tool calls and no queued messages remain.

### Message Flow Pipeline

```
AgentMessage[] → transformContext() → AgentMessage[] → convertToLlm() → Message[] → LLM
                    (optional)                           (required)
```

1. **transformContext**: Prune old messages, inject external context (operates on `AgentMessage[]`)
2. **convertToLlm**: Filter out non-LLM messages, convert custom types to standard `Message[]`

### Loop Structure

```
Outer loop (follow-up messages)
└── Inner loop (tool calls + steering messages)
    ├── Stream assistant response from LLM
    ├── If error/aborted → emit turn_end, agent_end, return
    ├── If tool calls present → execute tools
    ├── Emit turn_end
    ├── Poll for steering messages → if any, continue inner loop
    └── If no tool calls and no steering → exit inner loop
        └── Poll for follow-up messages → if any, continue outer loop
            └── If no follow-ups → emit agent_end, return
```

### Event Sequence: Simple Prompt

```
prompt("Hello")
├── agent_start
├── turn_start
├── message_start   { userMessage }
├── message_end     { userMessage }
├── message_start   { assistantMessage }
├── message_update  { partial... }          // streaming chunks
├── message_end     { assistantMessage }
├── turn_end        { message, toolResults: [] }
└── agent_end       { messages: [...] }
```

### Event Sequence: With Tool Calls

```
prompt("Read config.json")
├── agent_start
├── turn_start
├── message_start/end  { userMessage }
├── message_start      { assistantMessage with toolCall }
├── message_update...
├── message_end        { assistantMessage }
├── tool_execution_start  { toolCallId, toolName, args }
├── tool_execution_update { partialResult }           // if tool streams
├── tool_execution_end    { toolCallId, result }
├── message_start/end  { toolResultMessage }
├── turn_end           { message, toolResults: [...] }
│
├── turn_start                                         // next turn
├── message_start      { assistantMessage }
├── message_update...
├── message_end
├── turn_end
└── agent_end
```

---

## Tool Execution Pipeline

### Tool Execution Modes

**Parallel (default):** Tool calls from a single assistant message are preflighted sequentially (argument validation + `beforeToolCall` hook), then all allowed tools execute concurrently. Final `tool_execution_end` events and `toolResult` messages are emitted in the original assistant source order.

**Sequential:** Each tool call is prepared, executed, and finalized before the next one starts.

### AgentTool Interface

```typescript
interface AgentTool<TParameters extends TSchema = TSchema, TDetails = any> extends Tool<TParameters> {
  label: string;  // Human-readable name for UI display
  execute: (
    toolCallId: string,
    params: Static<TParameters>,
    signal?: AbortSignal,
    onUpdate?: AgentToolUpdateCallback<TDetails>,
  ) => Promise<AgentToolResult<TDetails>>;
}

interface AgentToolResult<T> {
  content: (TextContent | ImageContent)[];
  details: T;  // For UI display or logging
}
```

Tool parameters use TypeBox schemas (`@sinclair/typebox`). Arguments are validated with `validateToolArguments()` from pi-ai before execution.

### Error Handling

Tools should throw errors on failure (not return error content). The loop catches exceptions and reports them to the LLM as `toolResult` messages with `isError: true`.

### beforeToolCall Hook

Runs after `tool_execution_start` event and argument validation. Can block execution:

```typescript
beforeToolCall: async ({ toolCall, args, context, assistantMessage }, signal) => {
  if (toolCall.name === "bash") {
    return { block: true, reason: "bash is disabled" };
  }
  // return undefined to allow execution
}
```

When using the `Agent` class, the assistant `message_end` is processed as a barrier before tool preflight begins. This means `beforeToolCall` sees agent state that already includes the assistant message requesting the tool call.

### afterToolCall Hook

Runs after tool execution, before final tool events are emitted. Can override result fields:

```typescript
afterToolCall: async ({ toolCall, result, isError, context, assistantMessage }, signal) => {
  // Return partial override (field-by-field, no deep merge)
  return {
    content: [...],   // replaces content array in full
    details: {...},   // replaces details in full
    isError: false,   // overrides error flag
  };
}
```

Omitted fields keep their original values.

---

## Event System

### AgentEvent Types

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
  | { type: "tool_execution_end"; toolCallId: string; toolName: string; result: any; isError: boolean };
```

| Event | Emitted For | Key Data |
|-------|-------------|----------|
| `agent_start` | Agent begins processing | - |
| `agent_end` | Agent completes | All new messages |
| `turn_start` | New turn begins | - |
| `turn_end` | Turn completes | Assistant message + tool results |
| `message_start` | Any message begins | user, assistant, or toolResult |
| `message_update` | Assistant only, during streaming | Delta via `assistantMessageEvent` |
| `message_end` | Message completes | Final message |
| `tool_execution_start` | Tool begins | toolCallId, name, args |
| `tool_execution_update` | Tool streams progress | Partial result |
| `tool_execution_end` | Tool completes | Result, isError flag |

### State Updates During Events

The `Agent` class updates its state in response to events:

- `message_start` / `message_update` → updates `streamMessage`
- `message_end` → clears `streamMessage`, appends to `messages`
- `tool_execution_start` → adds to `pendingToolCalls`
- `tool_execution_end` → removes from `pendingToolCalls`
- `turn_end` → captures error from assistant message if present
- `agent_end` → sets `isStreaming = false`

---

## Custom Message Types

Extend `AgentMessage` via TypeScript declaration merging:

```typescript
declare module "@mariozechner/pi-agent-core" {
  interface CustomAgentMessages {
    notification: { role: "notification"; text: string; timestamp: number };
  }
}
```

Custom types must be handled in `convertToLlm` since LLMs only understand `user`, `assistant`, and `toolResult` roles.

---

## Steering and Follow-up Queues

### Steering Messages

Delivered after the current assistant turn finishes executing all its tool calls. The next LLM call sees the steering message in context.

- In `"one-at-a-time"` mode: one steering message per turn
- In `"all"` mode: all queued steering messages injected at once

### Follow-up Messages

Checked only when there are no more tool calls and no steering messages. If any are queued, they are injected and another turn runs.

### Queue Priority

```
1. Tool calls from assistant message (always complete)
2. Steering messages (interrupt between turns)
3. Follow-up messages (only when agent would otherwise stop)
```

---

## Transport Abstraction

The `transport` option (`"sse"`, `"websocket"`, `"auto"`) is forwarded to the underlying pi-ai stream function. Providers that support multiple transports can use this preference. Default is `"sse"`.

---

## Proxy Stream Function

For browser apps that route LLM calls through a backend server:

```typescript
import { Agent, streamProxy } from "@mariozechner/pi-agent-core";

const agent = new Agent({
  streamFn: (model, context, options) =>
    streamProxy(model, context, {
      ...options,
      authToken: "...",
      proxyUrl: "https://your-server.com",
    }),
});
```

The proxy server receives POST requests to `/api/stream` with model, context, and options. It returns Server-Sent Events with `ProxyAssistantMessageEvent` payloads (partial field stripped to reduce bandwidth). The client reconstructs the partial message locally.

### ProxyStreamOptions

```typescript
interface ProxyStreamOptions extends SimpleStreamOptions {
  authToken: string;   // Bearer token for proxy server
  proxyUrl: string;    // Server URL (e.g., "https://genai.example.com")
}
```

---

## Low-Level agentLoop() API

For direct control without the `Agent` class wrapper:

```typescript
import { agentLoop, agentLoopContinue } from "@mariozechner/pi-agent-core";

const context: AgentContext = {
  systemPrompt: "You are helpful.",
  messages: [],
  tools: [],
};

const config: AgentLoopConfig = {
  model: getModel("openai", "gpt-4o"),
  convertToLlm: (msgs) => msgs.filter(m =>
    ["user", "assistant", "toolResult"].includes(m.role)
  ),
  toolExecution: "parallel",
  beforeToolCall: async ({ toolCall, args }) => undefined,
  afterToolCall: async ({ toolCall, result, isError }) => undefined,
  getSteeringMessages: async () => [],
  getFollowUpMessages: async () => [],
};

// Start with prompt
for await (const event of agentLoop([userMessage], context, config)) {
  console.log(event.type);
}

// Continue from existing context
for await (const event of agentLoopContinue(context, config)) {
  console.log(event.type);
}
```

### Key Differences from Agent Class

The low-level streams are observational. They do **not** wait for your async event handling to settle before later producer phases continue. If you need `message_end` processing to act as a barrier before tool preflight (e.g., so `beforeToolCall` sees updated state), use the `Agent` class.

### AgentLoopConfig

Extends `SimpleStreamOptions` from pi-ai and adds:

| Field | Required | Description |
|-------|----------|-------------|
| `model` | yes | Model to use for LLM calls |
| `convertToLlm` | yes | Convert `AgentMessage[]` to `Message[]` |
| `transformContext` | no | Pre-transform messages before convertToLlm |
| `getApiKey` | no | Dynamic API key resolution |
| `getSteeringMessages` | no | Returns steering messages to inject mid-run |
| `getFollowUpMessages` | no | Returns follow-up messages after agent would stop |
| `toolExecution` | no | `"parallel"` (default) or `"sequential"` |
| `beforeToolCall` | no | Preflight hook |
| `afterToolCall` | no | Post-execution hook |

### Internal Run Functions

For imperative (non-streaming) usage, the package also exports:

- `runAgentLoop(prompts, context, config, emit, signal?, streamFn?)` - Returns `Promise<AgentMessage[]>`
- `runAgentLoopContinue(context, config, emit, signal?, streamFn?)` - Returns `Promise<AgentMessage[]>`

These take an `AgentEventSink` callback (`(event) => Promise<void> | void`) instead of returning an async iterable.

---

## Key Changelog Milestones

| Version | Date | Change |
|---------|------|--------|
| 0.60.0 | 2026-03-18 | Current release |
| 0.58.4 | 2026-03-16 | Fix: steering waits for full tool-call batch |
| 0.58.0 | 2026-03-14 | `beforeToolCall`/`afterToolCall` hooks; parallel tool execution mode |
| 0.52.12 | 2026-02-13 | `transport` option for provider transport preference |
| 0.50.8 | 2026-02-01 | `maxRetryDelayMs` option |
| 0.38.0 | 2026-01-08 | `thinkingBudgets` option |
| 0.37.3 | 2026-01-06 | `sessionId` for provider caching |
| 0.32.0 | 2026-01-03 | **Breaking:** `queueMessage()` split into `steer()`/`followUp()` |
| 0.31.0 | 2026-01-02 | **Breaking:** Transport abstraction removed; `streamFn` option; agent loop moved from pi-ai |
