# pi-agent-core (`@earendil-works/pi-agent-core`)

| Field | Value |
|-------|-------|
| Package | `@earendil-works/pi-agent-core` |
| Version | 0.74.0 (verified via `npm view @earendil-works/pi-agent-core version`) |
| Source | `packages/agent/` (commit `1eee081e`, tag `v0.74.0`) |
| License | MIT |
| Node | `>= 20.0.0` |
| Runtime deps | `@earendil-works/pi-ai ^0.74.0`, `typebox ^1.1.24` |

`pi-agent-core` is the agent runtime layer between `@earendil-works/pi-ai` (LLM provider abstraction) and downstream consumers such as `@earendil-works/pi-coding-agent` (the `pi` CLI). It exposes:

- A stateful `Agent` class that owns the transcript, emits lifecycle events, executes tools, and manages steering/follow-up queues.
- A pluggable stream function (`StreamFn`) that defaults to `streamSimple` from `pi-ai` and can be replaced (e.g. `streamProxy` for browser-to-server proxying).
- Low-level imperative entry points (`agentLoop`, `agentLoopContinue`, `runAgentLoop`, `runAgentLoopContinue`) that drive the same state machine without the `Agent` wrapper.

This package does **not** contain session persistence, compaction, skills, prompt templates, or any `AgentHarness` runtime — those live in `@earendil-works/pi-coding-agent`. The agent layer is intentionally pure: it owns in-memory state for one run and emits events; persistence is the caller's responsibility.

---

## Install

```bash
npm install @earendil-works/pi-agent-core @earendil-works/pi-ai
```

```ts
import { Agent, type AgentEvent } from "@earendil-works/pi-agent-core";
import { getModel } from "@earendil-works/pi-ai";

const agent = new Agent({
  initialState: {
    systemPrompt: "You are a helpful assistant.",
    model: getModel("anthropic", "claude-sonnet-4-20250514"),
  },
});

agent.subscribe(async (event: AgentEvent) => {
  if (event.type === "message_update" && event.assistantMessageEvent.type === "text_delta") {
    process.stdout.write(event.assistantMessageEvent.delta);
  }
});

await agent.prompt("Hello!");
```

---

## Public API surface

The barrel at `packages/agent/src/index.ts:1` re-exports four modules:

| Module | Path | Public symbols |
|--------|------|----------------|
| Agent class | `src/agent.ts` | `Agent` (`agent.ts:158`), `AgentOptions` (`agent.ts:94`) |
| Loop entry points | `src/agent-loop.ts` | `agentLoop` (`agent-loop.ts:31`), `agentLoopContinue` (`agent-loop.ts:64`), `runAgentLoop` (`agent-loop.ts:95`), `runAgentLoopContinue` (`agent-loop.ts:120`), `AgentEventSink` (`agent-loop.ts:25`) |
| Proxy stream | `src/proxy.ts` | `streamProxy` (`proxy.ts:116`), `ProxyAssistantMessageEvent` (`proxy.ts:36`), `ProxyStreamOptions` (`proxy.ts:73`) |
| Types | `src/types.ts` | `StreamFn`, `ToolExecutionMode`, `AgentToolCall`, `BeforeToolCallResult`/`Context`, `AfterToolCallResult`/`Context`, `ShouldStopAfterTurnContext`, `AgentLoopConfig`, `ThinkingLevel`, `CustomAgentMessages`, `AgentMessage`, `AgentState`, `AgentTool`, `AgentToolResult`, `AgentToolUpdateCallback`, `AgentContext`, `AgentEvent` |

All exports flow through `index.ts`; there are no deep imports.

---

## Agent class

### Constructor options (`AgentOptions`)

Defined at `src/agent.ts:94`. Every field is optional.

| Option | Type | Default | Purpose |
|--------|------|---------|---------|
| `initialState` | `Partial<AgentState>` (sans runtime fields) | see below | Seed `systemPrompt`, `model`, `thinkingLevel`, `tools`, `messages` |
| `convertToLlm` | `(msgs) => Message[] \| Promise<Message[]>` | filters to `user`/`assistant`/`toolResult` (`agent.ts:27`) | Convert `AgentMessage[]` to LLM-compatible `Message[]` at each turn |
| `transformContext` | `(msgs, signal?) => Promise<AgentMessage[]>` | none | Pre-`convertToLlm` hook for pruning / external context injection |
| `streamFn` | `StreamFn` | `streamSimple` from pi-ai | Replace the LLM stream (e.g. `streamProxy`) |
| `getApiKey` | `(provider) => Promise<string \| undefined> \| string \| undefined` | none | Resolve API keys per call (OAuth refresh) |
| `onPayload` | `SimpleStreamOptions["onPayload"]` | none | Inspect/replace provider payloads pre-send |
| `onResponse` | `SimpleStreamOptions["onResponse"]` | none | Inspect provider responses |
| `beforeToolCall` | hook (`agent.ts:170`) | none | Preflight; can block tool execution |
| `afterToolCall` | hook (`agent.ts:174`) | none | Post-execution; can override result fields |
| `steeringMode` | `"all" \| "one-at-a-time"` | `"one-at-a-time"` | Drain mode for the steering queue |
| `followUpMode` | `"all" \| "one-at-a-time"` | `"one-at-a-time"` | Drain mode for the follow-up queue |
| `sessionId` | `string` | none | Forwarded to providers for cache-aware backends |
| `thinkingBudgets` | `ThinkingBudgets` | none | Per-level token budgets (forwarded) |
| `transport` | `Transport` (`"sse" \| "websocket" \| "websocket-cached" \| "auto"`) | `"auto"` | Preferred transport, forwarded to the stream fn |
| `maxRetryDelayMs` | `number` | none (passthrough) | Cap on server-requested retry delays |
| `toolExecution` | `ToolExecutionMode` (`"parallel" \| "sequential"`) | `"parallel"` | How a batch of tool calls from one assistant message is run |

`Transport` is re-exported through `@earendil-works/pi-ai` and includes `"websocket-cached"` in v0.74.0 — confirmed at `packages/ai/src/types.ts:68`. Default became `"auto"` in v0.72.1.

### Default state

`createMutableAgentState()` at `agent.ts:64` builds:

```ts
{
  systemPrompt: "",
  model: DEFAULT_MODEL,        // synthetic { id:"unknown", api:"unknown", ... }
  thinkingLevel: "off",
  tools: [],                   // copied on assignment
  messages: [],                // copied on assignment
  isStreaming: false,
  streamingMessage: undefined,
  pendingToolCalls: new Set(),
  errorMessage: undefined,
}
```

If `initialState.model` is omitted, the agent will refuse useful work (the synthetic model has no real provider). Always pass a real `Model` from `getModel(...)`.

### State (`AgentState`)

Declared at `src/types.ts:288`. Use accessor properties for `tools` and `messages` — assigning replaces the top-level array with a **copy**:

```ts
agent.state.messages = newMessages; // top-level array copied
agent.state.messages.push(msg);     // mutates the internal array
```

Runtime-owned fields are readonly in the public type and cannot be passed to `initialState`:

- `isStreaming`
- `streamingMessage`
- `pendingToolCalls: ReadonlySet<string>`
- `errorMessage`

`isStreaming` stays `true` until every awaited `agent_end` listener has settled (`agent.ts:438`–`agent.ts:486`).

### Direct property mutators (v0.65.0+)

All `setXxx()` methods were removed in v0.65.0. Use direct assignment:

| Was | Now |
|-----|-----|
| `agent.setSystemPrompt(v)` | `agent.state.systemPrompt = v` |
| `agent.setModel(m)` | `agent.state.model = m` |
| `agent.setThinkingLevel(l)` | `agent.state.thinkingLevel = l` |
| `agent.setTools(t)` | `agent.state.tools = t` |
| `agent.replaceMessages(m)` | `agent.state.messages = m` |
| `agent.appendMessage(m)` | `agent.state.messages.push(m)` |
| `agent.clearMessages()` | `agent.state.messages = []` |
| `agent.setToolExecution(m)` | `agent.toolExecution = m` |
| `agent.setBeforeToolCall(fn)` | `agent.beforeToolCall = fn` |
| `agent.setAfterToolCall(fn)` | `agent.afterToolCall = fn` |
| `agent.setTransport(t)` | `agent.transport = t` |
| `agent.setSteeringMode(m)` | `agent.steeringMode = m` |
| `agent.getSteeringMode()` | `agent.steeringMode` |
| `agent.setFollowUpMode(m)` | `agent.followUpMode = m` |
| `agent.getFollowUpMode()` | `agent.followUpMode` |

`agent.reset()` (`agent.ts:302`) is unchanged and clears messages, runtime state, and both queues.

### Prompting and lifecycle

| Method | Signature | Notes |
|--------|-----------|-------|
| `prompt(text, images?)` | `agent.ts:313` | Wraps text + optional images into a user `AgentMessage` |
| `prompt(message)` | `agent.ts:313` | Pass a fully-formed `AgentMessage` |
| `prompt(messages)` | `agent.ts:313` | Pass an array; injected as the prompt batch |
| `continue()` | `agent.ts:326` | Resume from current transcript. If the tail is `assistant`, queued steering then follow-up messages are drained instead (`agent.ts:336`–`agent.ts:352`) |
| `abort()` | `agent.ts:288` | Aborts the active `AbortController` |
| `waitForIdle()` | `agent.ts:297` | Resolves after `agent_end` listeners settle and `finishRun()` clears runtime state |
| `reset()` | `agent.ts:302` | Clear transcript, runtime fields, and queues |

`prompt()` and `continue()` throw `"Agent is already processing..."` if invoked while `activeRun` is set (`agent.ts:316`, `agent.ts:327`). Use `steer()`/`followUp()` to inject messages during an active run.

### Steering and follow-up queues

| Method | Effect |
|--------|--------|
| `steer(msg)` (`agent.ts:252`) | Enqueue a message delivered **after the current assistant turn's full tool batch finishes**, before the next LLM call |
| `followUp(msg)` (`agent.ts:257`) | Enqueue a message delivered **only when the agent would otherwise stop** (no tool calls and no pending steering) |
| `clearSteeringQueue()` / `clearFollowUpQueue()` / `clearAllQueues()` | Drop queued messages |
| `hasQueuedMessages()` | True if either queue is non-empty |
| `agent.steeringMode` / `agent.followUpMode` | Get/set; `"all"` drains every queued message at once, `"one-at-a-time"` (default) drains the head only (`agent.ts:113`–`agent.ts:144`) |

### Subscribing to events

```ts
const unsubscribe = agent.subscribe(async (event, signal) => {
  if (event.type === "agent_end") {
    await persist(agent.state.messages, signal);
  }
});
```

Listeners are awaited in registration order (`agent.ts:539`). `signal` is the active turn's `AbortSignal` (also reachable as `agent.signal`, `agent.ts:283`).

`agent_end` is the **last loop event emitted** for a run, but the agent does not become idle until awaited `agent_end` listeners resolve and `finishRun()` runs. `waitForIdle()`, `prompt()`, and `continue()` only settle after that.

### Public properties

| Property | Type | Source |
|----------|------|--------|
| `state` | `AgentState` (readonly handle) | `agent.ts:229` |
| `signal` | `AbortSignal \| undefined` | `agent.ts:283` |
| `sessionId` | `string \| undefined` | `agent.ts:180` |
| `thinkingBudgets` | `ThinkingBudgets \| undefined` | `agent.ts:182` |
| `transport` | `Transport` | `agent.ts:184` |
| `maxRetryDelayMs` | `number \| undefined` | `agent.ts:186` |
| `toolExecution` | `ToolExecutionMode` | `agent.ts:188` |
| `streamFn` | `StreamFn` | `agent.ts:166` |
| `convertToLlm`, `transformContext`, `getApiKey`, `onPayload`, `onResponse`, `beforeToolCall`, `afterToolCall` | hooks | `agent.ts:164`–`agent.ts:177` |
| `steeringMode`, `followUpMode` | `"all" \| "one-at-a-time"` | `agent.ts:234`–`agent.ts:249` |

---

## Agent loop and state machine

### Message flow pipeline

```
AgentMessage[]  ── transformContext (optional) ──▶  AgentMessage[]
                ── convertToLlm   (required) ──▶  Message[]  ──▶  LLM
```

`transformContext` operates at the `AgentMessage` level (pruning, RAG injection). `convertToLlm` filters out anything that isn't a `user`, `assistant`, or `toolResult` and may map custom message types into LLM-understood roles. Both hooks have an explicit "must not throw or reject" contract (`types.ts:126`, `types.ts:155`).

The pipeline is applied at `streamAssistantResponse()` in `agent-loop.ts:252`–`agent-loop.ts:286`, once per LLM call.

### Loop structure (`runLoop`, `agent-loop.ts:155`)

```
┌─ Outer loop (follow-up replenishment)
│  pendingMessages := await getSteeringMessages?.() ?? []
│  ┌─ Inner loop (assistant turns + tool batches + steering)
│  │  emit turn_start (skipped on first iteration of first turn)
│  │  inject pendingMessages → append to context + emit message_start/end
│  │  message := streamAssistantResponse()
│  │  if message.stopReason ∈ {error, aborted}:
│  │       emit turn_end, agent_end → return
│  │  toolCalls := message.content where type == "toolCall"
│  │  if toolCalls.length > 0:
│  │       batch := executeToolCalls(...)
│  │       append batch.messages to context
│  │       hasMoreToolCalls := !batch.terminate
│  │  emit turn_end { message, toolResults }
│  │  if await shouldStopAfterTurn?.(...): emit agent_end → return
│  │  pendingMessages := await getSteeringMessages?.() ?? []
│  │  continue while (hasMoreToolCalls || pendingMessages.length > 0)
│  followUp := await getFollowUpMessages?.() ?? []
│  if followUp.length > 0: pendingMessages := followUp; continue outer
│  break
└─ emit agent_end { messages: newMessages }
```

Key transitions:

1. **Steering pre-poll.** Before the very first turn, `getSteeringMessages()` is polled (`agent-loop.ts:165`). When `Agent.continue()` resumes from queued steering after an assistant tail, the first poll is deliberately skipped using `skipInitialSteeringPoll` (`agent.ts:411`–`agent.ts:432`) — the queued messages are already the prompt batch.
2. **Prompts vs continue.** `agentLoop()` injects `prompts` as the initial pending batch and emits `message_start`/`message_end` for each (`agent-loop.ts:95`–`agent-loop.ts:118`). `agentLoopContinue()` skips that injection and emits only `agent_start` + `turn_start` (`agent-loop.ts:120`–`agent-loop.ts:143`). It also enforces that the last message is **not** assistant (`agent-loop.ts:131`).
3. **Error/aborted termination.** When the assistant message has `stopReason == "error" | "aborted"`, the loop emits `turn_end` with empty `toolResults` then `agent_end` and returns immediately (`agent-loop.ts:194`–`agent-loop.ts:198`). The `Agent` class additionally synthesises a failure assistant message if the executor throws (`agent.ts:463`–`agent.ts:478`).
4. **Tool-batch termination (v0.69.0).** `executeToolCalls()` returns `terminate: true` only when every finalized tool result in that batch sets `terminate: true` (`agent-loop.ts:511`). Mixed batches continue.
5. **`shouldStopAfterTurn` (v0.72.0).** Polled after `turn_end` and before steering/follow-up polls (`agent-loop.ts:218`). Returning `true` emits `agent_end` and exits without aborting the provider stream or running tools — it only suppresses the next LLM call.

### Event sequences

**Simple prompt:**

```
agent_start
turn_start
message_start { userMessage }
message_end   { userMessage }
message_start { assistantMessage }
message_update { partial, assistantMessageEvent }   // 0..N
message_end   { assistantMessage }
turn_end      { message: assistantMessage, toolResults: [] }
agent_end     { messages: [userMessage, assistantMessage] }
```

**Prompt with a tool call (sequential):**

```
agent_start
turn_start
message_start/end { userMessage }
message_start     { assistantMessage with toolCall }
message_update*
message_end       { assistantMessage }
tool_execution_start  { toolCallId, toolName, args }
tool_execution_update*                              // if execute() called onUpdate
tool_execution_end    { toolCallId, result, isError }
message_start/end { toolResultMessage }
turn_end          { message, toolResults: [toolResultMessage] }

turn_start
message_start     { assistantMessage }
message_update*
message_end
turn_end
agent_end
```

**Parallel batch with out-of-order completion** (verified by `agent-loop.test.ts:452`): `tool_execution_start` fires sequentially in source order during preflight; `tool_execution_end` fires in **completion** order; `message_start`/`message_end` for `toolResult` messages and `turn_end.toolResults` are emitted in **source** order. Fix from v0.68.1.

### `Agent` event-driven state reduction

`processEvents()` (`agent.ts:495`) maps loop events to mutable state:

| Event | Effect on `agent.state` |
|-------|-------------------------|
| `message_start`, `message_update` | `streamingMessage = event.message` |
| `message_end` | `streamingMessage = undefined`, push to `messages` |
| `tool_execution_start` | Add `toolCallId` to `pendingToolCalls` (new `Set` to keep `ReadonlySet` semantics) |
| `tool_execution_end` | Remove `toolCallId` from `pendingToolCalls` |
| `turn_end` | If `message.role === "assistant"` and `errorMessage` is present, copy to `state.errorMessage` |
| `agent_end` | `streamingMessage = undefined`; idle only after `finishRun()` |

When using the `Agent` class, `message_end` is processed as a barrier before tool preflight begins. That means `beforeToolCall` sees agent state that already contains the assistant message that requested the tool call. The low-level streams (`agentLoop`/`agentLoopContinue`) do not give this guarantee — they are observational and don't await your handlers before progressing to tool execution (`README.md:484`).

---

## Tool calling contract

### `AgentTool` (`types.ts:332`)

```ts
interface AgentTool<TParameters extends TSchema = TSchema, TDetails = any>
  extends Tool<TParameters> {
  // Inherited from Tool<TParameters> in pi-ai:
  //   name: string
  //   description: string
  //   parameters: TParameters

  label: string;
  prepareArguments?: (args: unknown) => Static<TParameters>;
  execute: (
    toolCallId: string,
    params: Static<TParameters>,
    signal?: AbortSignal,
    onUpdate?: AgentToolUpdateCallback<TDetails>,
  ) => Promise<AgentToolResult<TDetails>>;
  executionMode?: ToolExecutionMode;
}

interface AgentToolResult<T> {
  content: (TextContent | ImageContent)[];
  details: T;
  terminate?: boolean;
}
```

Parameter schemas use **TypeBox 1.x** — imported from the top-level `typebox` package (the v0.69.0 breaking change moved off `@sinclair/typebox`). Arguments are validated with `validateToolArguments()` from pi-ai before `execute()` (`agent-loop.ts:547`).

### `prepareArguments` (v0.64.0+)

`prepareArguments` runs **before** schema validation and can normalize legacy argument shapes (e.g. migrate top-level `oldText`/`newText` into an `edits[]` array). If it returns the same object reference, the tool call is passed through unchanged (`agent-loop.ts:515`–`agent-loop.ts:527`). Verified in `agent-loop.test.ts:372`.

### Execution modes

```ts
type ToolExecutionMode = "sequential" | "parallel";
```

- **Parallel (default).** `executeToolCallsParallel()` (`agent-loop.ts:424`) preflights every tool call sequentially (`prepareArguments` → validate → `beforeToolCall`), then launches allowed tools concurrently via `Promise.all`. `tool_execution_end` fires per tool in completion order, then `toolResult` messages and `turn_end.toolResults` are emitted in original source order.
- **Sequential.** `executeToolCallsSequential()` (`agent-loop.ts:372`) prepares, executes, and finalizes one tool at a time.

**Per-tool override (v0.67.x).** Setting `executionMode: "sequential"` on a single tool forces the entire batch to run sequentially, even if `toolExecution: "parallel"` is set globally (`agent-loop.ts:358`–`agent-loop.ts:363`). Verified in `agent-loop.test.ts:653` and `agent-loop.test.ts:736`. The inverse (one tool with `executionMode: "parallel"` inside a default-parallel config) is a no-op since parallel is already the default.

### Lifecycle hooks

`beforeToolCall(context, signal?)` (`types.ts:230`):

```ts
beforeToolCall: async ({ assistantMessage, toolCall, args, context }, signal) => {
  if (toolCall.name === "bash") {
    return { block: true, reason: "bash is disabled" };
  }
  return undefined; // allow execution
};
```

- Runs after schema validation but before `execute()`.
- Returning `{ block: true, reason }` short-circuits the tool, emitting an error tool result with `reason` (or `"Tool execution was blocked"` if omitted) and `isError: true` (`agent-loop.ts:558`).
- `args` is mutable; in-place mutation flows into `execute()` without re-validation (verified at `agent-loop.test.ts:310`).
- Validation/throw cases are caught and surfaced as error tool results (`agent-loop.ts:572`).

`afterToolCall(context, signal?)` (`types.ts:247`):

```ts
afterToolCall: async ({ assistantMessage, toolCall, args, result, isError, context }, signal) => {
  return {
    content: [...],   // replaces content array in full, no deep merge
    details: {...},   // replaces details
    isError: false,   // overrides error flag
    terminate: true,  // overrides early-termination hint
  };
};
```

- Runs **after** `execute()` resolves and **before** `tool_execution_end` is emitted.
- Field-by-field override; omitted fields preserve the executed values (`agent-loop.ts:629`–`agent-loop.ts:653`).
- A throw from `afterToolCall` is converted to an error tool result, **not** propagated, since v0.67.67 (fixes a batch-abort bug).

### Early termination (`terminate`, v0.69.0)

Setting `terminate: true` on every finalized tool result in a batch causes the loop to skip the automatic follow-up LLM call. Source: `shouldTerminateToolBatch()` (`agent-loop.ts:511`):

```ts
function shouldTerminateToolBatch(finalizedCalls) {
  return finalizedCalls.length > 0
    && finalizedCalls.every((f) => f.result.terminate === true);
}
```

Mixed batches (some terminate, some don't) continue normally. `afterToolCall` can also flip a tool result into a terminating one (verified at `agent-loop.test.ts:1111`).

### Error handling contract

Tools should **throw** on failure rather than returning error content. `executePreparedToolCall()` catches the throw and emits an `AgentToolResult` whose `content` is the error message (`agent-loop.ts:609`–`agent-loop.ts:615`); the resulting `ToolResultMessage` gets `isError: true`. Tools that swallow errors and return success content will be treated as successful.

### Tool result message shape

`createToolResultMessage()` (`agent-loop.ts:680`) emits:

```ts
{
  role: "toolResult",
  toolCallId,
  toolName,
  content,           // from AgentToolResult.content
  details,           // from AgentToolResult.details
  isError,
  timestamp: Date.now(),
}
```

This matches `ToolResultMessage` from pi-ai (`packages/ai/src/types.ts:235`).

---

## Transport abstraction

In v0.74.0, `pi-agent-core` has **no transport interface of its own** — the original `ProviderTransport`/`AppTransport`/`AgentTransport` interfaces were removed in v0.31.0. Pluggability now happens at two levels:

### 1. The `Transport` preference string

```ts
type Transport = "sse" | "websocket" | "websocket-cached" | "auto"; // packages/ai/src/types.ts:68
```

Set on `AgentOptions.transport` or `AgentLoopConfig.transport` (inherited from `SimpleStreamOptions`). It is forwarded into the stream function and lets each pi-ai provider pick an available transport. Default is `"auto"` (v0.72.1+).

### 2. The `StreamFn` callback

This is the actual extension point. `types.ts:24`:

```ts
type StreamFn = (
  ...args: Parameters<typeof streamSimple>
) => ReturnType<typeof streamSimple> | Promise<ReturnType<typeof streamSimple>>;
```

Contract (`types.ts:15`–`types.ts:23`):

> Must not throw or return a rejected promise for request/model/runtime failures. Must return an `AssistantMessageEventStream`. Failures must be encoded in the returned stream via protocol events and a final `AssistantMessage` with `stopReason` `"error"` or `"aborted"` and `errorMessage`.

The default `streamFn` is `streamSimple` from pi-ai. Replacing it lets the agent talk to:

- A proxy server (`streamProxy` in this package).
- A test mock (e.g. `MockAssistantStream` in the agent test suite — `test/agent.test.ts:6`).
- Faux providers (`registerFauxProvider` from pi-ai, used in `test/e2e.test.ts`).
- Any custom backend that produces an `AssistantMessageEventStream`.

The `Agent` class forwards the configured `streamFn` (plus `apiKey` resolved via `getApiKey`) into `runAgentLoop` (`agent.ts:374`–`agent.ts:399`).

### `streamProxy` — proxy stream function

For browser apps that route LLM calls through a server (auth, secret management, observability):

```ts
import { Agent, streamProxy } from "@earendil-works/pi-agent-core";

const agent = new Agent({
  streamFn: (model, context, options) =>
    streamProxy(model, context, {
      ...options,
      authToken: await getAuthToken(),
      proxyUrl: "https://genai.example.com",
    }),
});
```

`streamProxy` (`proxy.ts:116`) POSTs to `${proxyUrl}/api/stream` with `{ model, context, options }` (only a serializable subset of options — `proxy.ts:101`). The server returns SSE lines (`data: <json>\n`) whose payloads are `ProxyAssistantMessageEvent` records.

The client reconstructs the running `AssistantMessage` locally because `partial` is stripped at the wire to reduce bandwidth (`processProxyEvent`, `proxy.ts:238`–`proxy.ts:367`).

#### `ProxyStreamOptions`

```ts
interface ProxyStreamOptions extends Pick<SimpleStreamOptions,
  | "temperature"
  | "maxTokens"
  | "reasoning"
  | "cacheRetention"
  | "sessionId"
  | "headers"
  | "metadata"
  | "transport"
  | "thinkingBudgets"
  | "maxRetryDelayMs"
> {
  signal?: AbortSignal;
  authToken: string;
  proxyUrl: string;
}
```

The v0.68.1 fix expanded the serializable subset to include session, transport, retry-delay, metadata, header, cache-retention, and thinking-budget settings.

#### `ProxyAssistantMessageEvent`

The wire format the server emits (`proxy.ts:36`):

```ts
type ProxyAssistantMessageEvent =
  | { type: "start" }
  | { type: "text_start";    contentIndex: number }
  | { type: "text_delta";    contentIndex: number; delta: string }
  | { type: "text_end";      contentIndex: number; contentSignature?: string }
  | { type: "thinking_start"; contentIndex: number }
  | { type: "thinking_delta"; contentIndex: number; delta: string }
  | { type: "thinking_end";   contentIndex: number; contentSignature?: string }
  | { type: "toolcall_start"; contentIndex: number; id: string; toolName: string }
  | { type: "toolcall_delta"; contentIndex: number; delta: string }
  | { type: "toolcall_end";   contentIndex: number }
  | { type: "done";  reason: "stop" | "length" | "toolUse"; usage }
  | { type: "error"; reason: "aborted" | "error"; errorMessage?: string; usage };
```

Each event mutates a client-side `partial: AssistantMessage` and is re-emitted as a full `AssistantMessageEvent` (with `partial`) into the local stream.

---

## Attachments

`pi-agent-core` does not have a dedicated attachments API. The `UserMessageWithAttachments` and `Attachment` types were removed in v0.31.0; attachment handling is the **caller's responsibility** via the `convertToLlm` function and the standard `ImageContent` block type.

In practice:

```ts
await agent.prompt("What's in this image?", [
  { type: "image", data: base64Data, mimeType: "image/jpeg" },
]);
```

`prompt(input, images?)` wraps the text and any provided images into a single user message with mixed `TextContent`/`ImageContent` blocks (`agent.ts:355`–`agent.ts:372`). For richer attachment types (PDFs, file paths, custom payloads), declare a custom message via `CustomAgentMessages` and translate it in `convertToLlm` before the LLM sees it.

`ToolResultMessage.content` also accepts `ImageContent`, so tools can return images directly (`types.ts:316`).

---

## State management

This layer holds **in-memory state for one Agent instance** and nothing else:

- The transcript (`state.messages`) — appended in-place during streaming.
- The active prompt/turn lifecycle (`isStreaming`, `streamingMessage`, `pendingToolCalls`, `errorMessage`).
- The current `model`, `systemPrompt`, `thinkingLevel`, `tools`, `sessionId`, transport, etc.
- The steering and follow-up queues.

There is **no built-in persistence, compaction, session store, skill registry, prompt template engine, execution environment, or resource manager at this layer.** Those are layered on top by `@earendil-works/pi-coding-agent` (the `AgentHarness`). For session-level concerns from inside `pi-agent-core`, consumers typically:

- `agent.subscribe(...)` for `message_end`, `turn_end`, or `agent_end` and persist the snapshot of `agent.state.messages`.
- Use `transformContext` to compact / prune before each LLM call.
- Use `shouldStopAfterTurn` to exit gracefully before context overflows.
- Use `sessionId` to opt into provider-side prompt caching (Anthropic prompt caching, etc.).

The previous version of this document warned about unreleased `AgentHarness` exports on `main`; in v0.74.0 those symbols **do not exist** in `packages/agent/src/`. Verified by `grep -r "AgentHarness\|SessionRepository\|compaction" packages/agent/src/` returning no matches.

---

## Types reference

Quick map of every public type, with the canonical declaration site.

### Configuration

| Type | Location | Notes |
|------|----------|-------|
| `AgentOptions` | `agent.ts:94` | Constructor input for `Agent` |
| `AgentLoopConfig` | `types.ts:115` | Extends `SimpleStreamOptions`; required for `agentLoop`/`agentLoopContinue` |
| `StreamFn` | `types.ts:24` | Pluggable LLM stream — must not throw |
| `ToolExecutionMode` | `types.ts:36` | `"sequential" \| "parallel"` |
| `ThinkingLevel` | `types.ts:255` | `"off" \| "minimal" \| "low" \| "medium" \| "high" \| "xhigh"` |

### Messages and content

| Type | Location | Notes |
|------|----------|-------|
| `AgentMessage` | `types.ts:280` | `Message \| CustomAgentMessages[keyof CustomAgentMessages]` |
| `CustomAgentMessages` | `types.ts:271` | Extension point via declaration merging |
| `AgentContext` | `types.ts:358` | `{ systemPrompt, messages, tools? }` |

### Tools

| Type | Location | Notes |
|------|----------|-------|
| `AgentTool<TParameters, TDetails>` | `types.ts:332` | TypeBox 1.x schema |
| `AgentToolResult<T>` | `types.ts:316` | `{ content, details, terminate? }` |
| `AgentToolUpdateCallback<T>` | `types.ts:329` | Passed to `tool.execute(_,_,_,onUpdate)` |
| `AgentToolCall` | `types.ts:39` | Convenience alias for the assistant tool-call content block |

### Hook contexts and results

| Type | Location |
|------|----------|
| `BeforeToolCallContext` | `types.ts:76` |
| `BeforeToolCallResult` | `types.ts:47` |
| `AfterToolCallContext` | `types.ts:88` |
| `AfterToolCallResult` | `types.ts:64` |
| `ShouldStopAfterTurnContext` | `types.ts:104` |

### State and events

| Type | Location |
|------|----------|
| `AgentState` | `types.ts:288` |
| `AgentEvent` | `types.ts:374` |

`AgentEvent` (verbatim, `types.ts:374`–`types.ts:389`):

```ts
type AgentEvent =
  | { type: "agent_start" }
  | { type: "agent_end"; messages: AgentMessage[] }
  | { type: "turn_start" }
  | { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
  | { type: "message_start"; message: AgentMessage }
  | { type: "message_update";
      message: AgentMessage;
      assistantMessageEvent: AssistantMessageEvent }
  | { type: "message_end"; message: AgentMessage }
  | { type: "tool_execution_start";
      toolCallId: string; toolName: string; args: any }
  | { type: "tool_execution_update";
      toolCallId: string; toolName: string; args: any; partialResult: any }
  | { type: "tool_execution_end";
      toolCallId: string; toolName: string; result: any; isError: boolean };
```

### Proxy

| Type | Location |
|------|----------|
| `ProxyAssistantMessageEvent` | `proxy.ts:36` |
| `ProxyStreamOptions` | `proxy.ts:73` |

---

## Low-level loop API

For direct control without the `Agent` wrapper, use `agentLoop`/`agentLoopContinue` (async iterables) or the imperative `runAgentLoop`/`runAgentLoopContinue` (callback-based).

```ts
import {
  agentLoop,
  agentLoopContinue,
  runAgentLoop,
  type AgentContext,
  type AgentLoopConfig,
  type AgentEventSink,
} from "@earendil-works/pi-agent-core";
import { getModel } from "@earendil-works/pi-ai";

const context: AgentContext = {
  systemPrompt: "You are helpful.",
  messages: [],
  tools: [],
};

const config: AgentLoopConfig = {
  model: getModel("openai", "gpt-4o"),
  convertToLlm: (msgs) => msgs.filter(
    (m) => m.role === "user" || m.role === "assistant" || m.role === "toolResult",
  ),
  toolExecution: "parallel",
  // beforeToolCall, afterToolCall, getSteeringMessages,
  // getFollowUpMessages, shouldStopAfterTurn, transformContext, getApiKey
};

const userMessage = {
  role: "user" as const,
  content: "Hello",
  timestamp: Date.now(),
};

// Iterable form:
for await (const event of agentLoop([userMessage], context, config)) {
  console.log(event.type);
}

// Callback form:
const emit: AgentEventSink = (event) => console.log(event.type);
const finalMessages = await runAgentLoop([userMessage], context, config, emit);
```

### `AgentLoopConfig` (`types.ts:115`)

Extends `SimpleStreamOptions` from pi-ai (so all stream-level options — `apiKey`, `temperature`, `maxTokens`, `reasoning`, `cacheRetention`, `sessionId`, `headers`, `metadata`, `transport`, `thinkingBudgets`, `maxRetryDelayMs`, `onPayload`, `onResponse`, `signal` — are valid).

| Field | Required | Purpose |
|-------|----------|---------|
| `model` | yes | Model to use for LLM calls |
| `convertToLlm` | yes | `AgentMessage[] → Message[]` |
| `transformContext` | no | Pre-`convertToLlm` AgentMessage transform |
| `getApiKey` | no | Per-call API key resolver (OAuth refresh) |
| `getSteeringMessages` | no | Polled before each new turn |
| `getFollowUpMessages` | no | Polled when the agent would otherwise stop |
| `toolExecution` | no | Default `"parallel"` |
| `beforeToolCall` | no | Preflight, can block |
| `afterToolCall` | no | Post-execution mutation |
| `shouldStopAfterTurn` | no | v0.72.0+; graceful exit after `turn_end` |

### Differences from the `Agent` class

The low-level streams are **observational**. They do not await your async event handling before later producer phases continue. If you need `message_end` to act as a barrier before tool preflight (so `beforeToolCall` sees the updated transcript), use the `Agent` class instead. This is called out at `README.md:484`.

`runAgentLoop` / `runAgentLoopContinue` (`agent-loop.ts:95`, `agent-loop.ts:120`) accept an `AgentEventSink` (`(event) => Promise<void> | void`) and return `Promise<AgentMessage[]>` — the array of **new** messages produced in that run. `runAgentLoopContinue` returns only the messages it produces; it does not include the pre-existing context (`agent-loop.ts:135`).

---

## Examples (from the test suite)

### Custom message types via declaration merging

```ts
declare module "@earendil-works/pi-agent-core" {
  interface CustomAgentMessages {
    notification: { role: "notification"; text: string; timestamp: number };
  }
}

const agent = new Agent({
  convertToLlm: (messages) =>
    messages.flatMap((m) => {
      if (m.role === "notification") return []; // hide from LLM
      return [m];
    }),
});
```

Verified by `agent-loop.test.ts:131` (custom messages survive in context but get filtered before the LLM).

### Context pruning

```ts
const agent = new Agent({
  transformContext: async (messages) => messages.slice(-20),
});
```

Verified by `agent-loop.test.ts:186`.

### Per-tool sequential override

```ts
const editTool: AgentTool<typeof schema> = {
  name: "edit",
  label: "Edit",
  description: "...",
  parameters: schema,
  executionMode: "sequential", // forces whole batch sequential
  execute: async (id, params) => ({ content: [...], details: {} }),
};
```

Verified by `agent-loop.test.ts:653`. Even with `toolExecution: "parallel"` (the default), the batch is forced sequential.

### `prepareArguments` for legacy schema migration

```ts
const editTool: AgentTool<typeof schema> = {
  // ...
  prepareArguments: (raw) => {
    const input = raw as { edits?: any[]; oldText?: string; newText?: string };
    if (typeof input.oldText === "string" && typeof input.newText === "string") {
      return { edits: [...(input.edits ?? []), { oldText: input.oldText, newText: input.newText }] };
    }
    return raw as { edits: any[] };
  },
  execute: async (_, params) => ({ content: [...], details: { count: params.edits.length } }),
};
```

Verified by `agent-loop.test.ts:372`.

### Steering during a tool batch

```ts
agent.steer({
  role: "user",
  content: [{ type: "text", text: "Stop! Try this approach instead." }],
  timestamp: Date.now(),
});
```

The steering message is delivered **only after the current assistant message's full tool batch finishes** (regression test: `agent-loop.test.ts:547`; fix from v0.58.4).

### Calculator tool (test fixture, `test/utils/calculate.ts:24`)

```ts
import { Type, type Static } from "typebox";
import type { AgentTool } from "@earendil-works/pi-agent-core";

const calculateSchema = Type.Object({
  expression: Type.String({ description: "The mathematical expression to evaluate" }),
});

export const calculateTool: AgentTool<typeof calculateSchema, undefined> = {
  label: "Calculator",
  name: "calculate",
  description: "Evaluate mathematical expressions",
  parameters: calculateSchema,
  execute: async (_toolCallId, args) => {
    const result = new Function(`return ${args.expression}`)();
    return {
      content: [{ type: "text", text: `${args.expression} = ${result}` }],
      details: undefined,
    };
  },
};
```

E2E usage with the pi-ai faux provider lives in `test/e2e.test.ts:60`.

### `shouldStopAfterTurn` graceful exit

```ts
const stream = agentLoop(prompts, context, {
  model,
  convertToLlm,
  shouldStopAfterTurn: async ({ context, newMessages }) =>
    estimateTokens(context.messages) > LIMIT,
});
```

Verified by `agent-loop.test.ts:897`: after returning `true`, the loop emits exactly `agent_start, turn_start, …, turn_end, agent_end` with no further LLM call, and queued follow-up messages are **not** polled.

### Forwarding the abort signal into tool calls

`agent.signal` exposes the active turn's `AbortSignal`. Inside `execute()` the same signal is the third parameter:

```ts
execute: async (_id, params, signal) => {
  const res = await fetch(params.url, { signal });
  return { content: [{ type: "text", text: await res.text() }], details: {} };
};
```

`agent.abort()` cancels the controller; tools must honor `signal` themselves.

---

## DeepWiki stale-claim callouts

DeepWiki indexed the repo at commit `156a90` (April 30 2026). Several claims are stale or slightly imprecise versus v0.74.0:

- **Package scope.** DeepWiki names the package `@mariozechner/pi-agent-core`. In v0.74.0 it is `@earendil-works/pi-agent-core` (lockstep scope migration, v0.74.0 changelog entry).
- **Transport literals.** DeepWiki's transport-abstraction page lists `Transport` as `"sse" | "rest"`. The actual type is `"sse" | "websocket" | "websocket-cached" | "auto"` (`packages/ai/src/types.ts:68`) and the default is `"auto"` (v0.72.1+), not `"sse"`.
- **`AgentMessage` shape.** DeepWiki says `AgentMessage` requires `id` and `timestamp` fields. There is no `id` field on `AgentMessage`; it is `Message | CustomAgentMessages[keyof CustomAgentMessages]` (`types.ts:280`), and only `timestamp` is part of the underlying pi-ai message types. Custom message types may or may not have `id`.
- **Transport interfaces.** Older DeepWiki sections may reference `ProviderTransport`/`AppTransport`/`AgentTransport` as injectable interfaces. These were **removed in v0.31.0**. Replace them with the `streamFn` callback.
- **Attachment types.** Any reference to `UserMessageWithAttachments` or a top-level `Attachment` type is stale; those were removed in v0.31.0. Attachments now flow as `ImageContent` blocks inside a normal user message, or via custom `AgentMessage` types translated by `convertToLlm`.
- **Parallel tool ordering.** DeepWiki summaries claim parallel mode emits "tool result messages re-ordered to match the assistant's original source order" without distinguishing from `tool_execution_end`. In v0.74.0, `tool_execution_end` follows **completion order**; only `toolResult` messages and `turn_end.toolResults` follow source order (v0.68.1 fix, verified by `agent-loop.test.ts:452`).
- **`shouldStopAfterTurn`, `terminate`, per-tool `executionMode`.** DeepWiki does not mention these. All are present in v0.74.0 (`shouldStopAfterTurn` v0.72.0; `terminate` v0.69.0; per-tool `executionMode` v0.67.x).
- **State machine specifics.** DeepWiki's section 3.1 describes generic "phases" but does not document the inner/outer loop, the steering pre-poll, the `skipInitialSteeringPoll` flag, or the `terminate`/`shouldStopAfterTurn` early-exit branches. The source-of-truth implementation is `runLoop()` at `packages/agent/src/agent-loop.ts:155`.

---

## Key changelog milestones

| Version | Date | Change |
|---------|------|--------|
| 0.74.0 | 2026-05-07 | Lockstep release: scope migration to `@earendil-works/*` |
| 0.73.1 | 2026-05-07 | Lockstep release |
| 0.73.0 | 2026-05-04 | Lockstep release |
| 0.72.1 | 2026-05-02 | Default `transport` changed from `"sse"` to `"auto"` |
| 0.72.0 | 2026-05-01 | `shouldStopAfterTurn` callback in `AgentLoopConfig` |
| 0.69.0 | 2026-04-22 | **Breaking:** TypeBox 1.x migration; `terminate: true` tool-result hint |
| 0.68.1 | 2026-04-22 | Fix: parallel `tool_execution_end` fires in completion order; `streamProxy` preserves full serializable option set |
| 0.67.67 | 2026-04-17 | Fix: parallel `afterToolCall` throws no longer abort the batch |
| 0.67.x | 2026-04-13–16 | Per-tool `executionMode` override |
| 0.65.0 | 2026-04-03 | **Breaking:** `AgentState` reshape; `streamMessage`→`streamingMessage`, `error`→`errorMessage`; `setXxx()` methods removed; async `subscribe` listeners with `AbortSignal`; `agent_end` becomes last loop event |
| 0.64.0 | 2026-03-29 | `AgentTool.prepareArguments` hook |
| 0.63.2 | 2026-03-29 | `Agent.signal` exposes active turn `AbortSignal` |
| 0.58.4 | 2026-03-16 | Fix: steering waits for full tool-call batch |
| 0.58.0 | 2026-03-14 | `beforeToolCall`/`afterToolCall` hooks; parallel tool execution |
| 0.52.12 | 2026-02-13 | `transport` option forwarding |
| 0.52.7 | 2026-02-06 | Fix: `continue()` resumes queued steering/follow-up when transcript ends in assistant |
| 0.50.8 | 2026-02-01 | `maxRetryDelayMs` option |
| 0.38.0 | 2026-01-08 | `thinkingBudgets` option |
| 0.37.3 | 2026-01-06 | `sessionId` option |
| 0.32.0 | 2026-01-03 | **Breaking:** `queueMessage()` split into `steer()` / `followUp()` |
| 0.31.0 | 2026-01-02 | **Breaking:** Transport interfaces removed; `streamFn` introduced; agent loop moved from pi-ai; `streamProxy` added; `UserMessageWithAttachments` removed |

---

## References

- Source files (commit `1eee081e`, tag `v0.74.0`):
  - `packages/agent/src/index.ts`
  - `packages/agent/src/agent.ts`
  - `packages/agent/src/agent-loop.ts`
  - `packages/agent/src/proxy.ts`
  - `packages/agent/src/types.ts`
- Tests:
  - `packages/agent/test/agent.test.ts`
  - `packages/agent/test/agent-loop.test.ts`
  - `packages/agent/test/e2e.test.ts`
  - `packages/agent/test/utils/calculate.ts`
- Package metadata: `packages/agent/package.json`, `packages/agent/CHANGELOG.md`, `packages/agent/README.md`
- Adjacent docs:
  - `02-pi-ai-llm-abstraction.md` — the LLM layer providing `streamSimple`, `Message`, `Tool`, `Model`, `Transport`
  - `04-pi-coding-agent.md` — the higher-level harness (`AgentHarness`, sessions, skills, prompts) that consumes this package
  - `08-sessions-and-persistence.md` — session persistence patterns
- DeepWiki:
  - https://deepwiki.com/badlogic/pi-mono/3-pi-agent-core:-agent-framework
  - https://deepwiki.com/badlogic/pi-mono/3.1-agent-loop-and-state-machine
  - https://deepwiki.com/badlogic/pi-mono/3.2-transport-abstraction-and-types
