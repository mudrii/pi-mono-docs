# @mariozechner/pi-ai

**Version:** 0.58.3 · **License:** MIT · [npm](https://www.npmjs.com/package/@mariozechner/pi-ai)

Unified multi-provider LLM streaming API. Provides a consistent interface over 23+ providers with automatic model discovery, token/cost tracking, streaming tool calls, and OAuth utilities.

---

## Installation

```bash
npm install @mariozechner/pi-ai
```

---

## Quick Start

```typescript
import { getModel, streamSimple } from "@mariozechner/pi-ai";

const model = getModel("anthropic", "claude-opus-4-6");

const context = {
  systemPrompt: "You are a helpful assistant.",
  messages: [
    {
      role: "user",
      content: "What is TypeScript?",
      timestamp: Date.now()
    }
  ]
};

const stream = streamSimple(model, context, { reasoning: "medium" });

for await (const event of stream) {
  if (event.type === "text_delta") {
    process.stdout.write(event.delta);
  }
  if (event.type === "done") {
    console.log("\nTokens:", event.message.usage.totalTokens);
  }
}
```

---

## Model Discovery

```typescript
import { getModel, getModels, getProviders } from "@mariozechner/pi-ai";

// Get a specific model (type-safe)
const model = getModel("anthropic", "claude-opus-4-6");

// List all models for a provider
const anthropicModels = getModels("anthropic");

// List all providers
const providers = getProviders();
```

### Model Interface

```typescript
interface Model<TApi extends Api> {
  id: string;
  name: string;
  api: Api;               // Wire protocol ID
  provider: Provider;     // Provider ID
  baseUrl: string;
  reasoning: boolean;     // Supports thinking/reasoning
  input: ("text" | "image")[];
  cost: {
    input: number;        // $/million tokens
    output: number;
    cacheRead: number;
    cacheWrite: number;
  };
  contextWindow: number;
  maxTokens: number;
}
```

---

## Streaming

### `streamSimple()` — Recommended

```typescript
import { streamSimple } from "@mariozechner/pi-ai";
import type { SimpleStreamOptions } from "@mariozechner/pi-ai";

const options: SimpleStreamOptions = {
  reasoning: "medium",        // "minimal" | "low" | "medium" | "high" | "xhigh"
  temperature: 0.7,
  maxTokens: 4096,
  signal: abortController.signal
};

const stream = streamSimple(model, context, options);

for await (const event of stream) {
  switch (event.type) {
    case "text_delta":
      process.stdout.write(event.delta);
      break;
    case "thinking_delta":
      process.stdout.write(`[thinking: ${event.delta}]`);
      break;
    case "toolcall_end":
      console.log("Tool call:", event.toolCall.name, event.toolCall.arguments);
      break;
    case "done":
      console.log("Stop reason:", event.reason);
      break;
    case "error":
      console.error("Error:", event.error.errorMessage);
      break;
  }
}

// Or get the final message directly
const message = await stream.result();
```

### `stream()` — Provider-Specific Options

```typescript
import { stream } from "@mariozechner/pi-ai";

const streamObj = stream(model, context, providerSpecificOptions);
```

### Stream Events

```typescript
type AssistantMessageEvent =
  | { type: "start"; partial: AssistantMessage }
  | { type: "text_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "text_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "text_end"; contentIndex: number; content: string; partial: AssistantMessage }
  | { type: "thinking_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "thinking_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "thinking_end"; contentIndex: number; content: string; partial: AssistantMessage }
  | { type: "toolcall_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "toolcall_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "toolcall_end"; contentIndex: number; toolCall: ToolCall; partial: AssistantMessage }
  | { type: "done"; reason: "stop" | "length" | "toolUse"; message: AssistantMessage }
  | { type: "error"; reason: "aborted" | "error"; error: AssistantMessage };
```

---

## Completion (Non-Streaming)

```typescript
import { completeSimple, complete } from "@mariozechner/pi-ai";

// Simple completion
const message = await completeSimple(model, context, options);
console.log(message.content);

// Provider-specific options
const message2 = await complete(model, context, providerOptions);
```

---

## Message Types

```typescript
// User message
interface UserMessage {
  role: "user";
  content: string | (TextContent | ImageContent)[];
  timestamp: number;
}

// Assistant response
interface AssistantMessage {
  role: "assistant";
  content: (TextContent | ThinkingContent | ToolCall)[];
  api: Api;
  provider: Provider;
  model: string;
  usage: Usage;
  stopReason: StopReason;
  errorMessage?: string;
  timestamp: number;
}

// Tool result
interface ToolResultMessage<TDetails = any> {
  role: "toolResult";
  toolCallId: string;
  toolName: string;
  content: (TextContent | ImageContent)[];
  details?: TDetails;
  isError: boolean;
  timestamp: number;
}

// Content blocks
interface TextContent { type: "text"; text: string }
interface ThinkingContent { type: "thinking"; thinking: string; redacted?: boolean }
interface ImageContent { type: "image"; data: string; mimeType: string }
interface ToolCall { type: "toolCall"; id: string; name: string; arguments: Record<string, any> }
```

---

## Tool Calling

Define tools using TypeBox schemas:

```typescript
import { Type } from "@mariozechner/pi-ai";  // Re-exported from @sinclair/typebox

const tool = {
  name: "get_weather",
  description: "Get weather for a city",
  parameters: Type.Object({
    city: Type.String({ description: "City name" }),
    units: Type.Optional(Type.Union([
      Type.Literal("celsius"),
      Type.Literal("fahrenheit")
    ]))
  })
};

const context = {
  messages: [...],
  tools: [tool]
};

// Stream with tool calling
for await (const event of streamSimple(model, context)) {
  if (event.type === "toolcall_end") {
    const { id, name, arguments: args } = event.toolCall;

    // Execute tool and add result to context
    const result = await executeMyTool(args);
    context.messages.push({
      role: "toolResult",
      toolCallId: id,
      toolName: name,
      content: [{ type: "text", text: result }],
      isError: false,
      timestamp: Date.now()
    });
  }
}
```

---

## Token Usage and Costs

```typescript
const message = await completeSimple(model, context);

console.log(message.usage);
// {
//   input: 1234,
//   output: 567,
//   cacheRead: 200,
//   cacheWrite: 800,
//   totalTokens: 2801,
//   cost: {
//     input: 0.001234,
//     output: 0.001701,
//     cacheRead: 0.00002,
//     cacheWrite: 0.002,
//     total: 0.004955
//   }
// }

// Calculate cost separately
import { calculateCost } from "@mariozechner/pi-ai";
const cost = calculateCost(model, usage);
```

---

## Context Structure

```typescript
interface Context {
  systemPrompt?: string;
  messages: Message[];
  tools?: Tool[];
}
```

---

## Stream Options

```typescript
interface SimpleStreamOptions {
  reasoning?: ThinkingLevel;          // "minimal" | "low" | "medium" | "high" | "xhigh"
  thinkingBudgets?: ThinkingBudgets;  // Per-level token budgets
  temperature?: number;
  maxTokens?: number;
  signal?: AbortSignal;
  apiKey?: string;
  transport?: "sse" | "websocket" | "auto";
  cacheRetention?: "none" | "short" | "long";
  sessionId?: string;
  onPayload?: (payload: unknown, model: Model<Api>) => unknown | Promise<unknown>;
  headers?: Record<string, string>;
  maxRetryDelayMs?: number;
  metadata?: Record<string, unknown>;
}
```

---

## Abort / Cancel

```typescript
const controller = new AbortController();

const stream = streamSimple(model, context, {
  signal: controller.signal
});

// Cancel after 5 seconds
setTimeout(() => controller.abort(), 5000);

for await (const event of stream) {
  if (event.type === "error" && event.reason === "aborted") {
    console.log("Cancelled");
    break;
  }
}
```

---

## OAuth

```typescript
import { login, refresh, getToken } from "@mariozechner/pi-ai/oauth";

// Login to a provider
await login("anthropic");    // Opens browser
await login("github-copilot");
await login("google-gemini-cli");

// Get current token
const token = await getToken("anthropic");

// Refresh token
await refresh("anthropic");
```

---

## Proxy Support

```typescript
import { streamProxy } from "@mariozechner/pi-agent-core";

const stream = streamProxy(model, context, {
  authToken: "bearer-token",
  proxyUrl: "https://your-proxy.example.com/api/stream"
});
```

---

## Custom Provider Registration

```typescript
import { registerApiProvider, clearApiProviders } from "@mariozechner/pi-ai";

registerApiProvider({
  api: "openai-completions",
  stream: myCustomStream,
  streamSimple: myCustomStreamSimple
}, "my-source-id");

// Later remove
// clearApiProviders() or unregisterApiProviders("my-source-id")
```

---

## Utilities

```typescript
import {
  calculateCost,
  supportsXhigh,
  modelsAreEqual,
  parseStreamingJson,
  validateToolArguments,
  checkContextOverflow,
  sanitizeSurrogates
} from "@mariozechner/pi-ai";

// Check if model supports xhigh reasoning
if (supportsXhigh(model)) {
  // Use xhigh thinking level
}

// Check if context is near overflow
if (checkContextOverflow(context, model)) {
  // Compact messages
}

// Parse partial JSON from streaming tool args
const partial = parseStreamingJson('{"city": "Lon');
// Returns best-effort parsed object
```

---

## Supported Providers (23+)

`amazon-bedrock`, `anthropic`, `azure-openai-responses`, `cerebras`, `github-copilot`, `google`, `google-antigravity`, `google-gemini-cli`, `google-vertex`, `groq`, `huggingface`, `kimi-coding`, `minimax`, `minimax-cn`, `mistral`, `openai`, `openai-codex`, `opencode`, `opencode-go`, `openrouter`, `vercel-ai-gateway`, `xai`, `zai`
