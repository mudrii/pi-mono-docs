# pi-ai: Unified LLM Abstraction Layer

`@mariozechner/pi-ai` (v0.65.2) is a unified LLM API library that provides automatic model discovery, provider configuration, token and cost tracking, and context persistence across 23 providers. It is the foundational AI layer of the Pi-Mono project and is published independently on npm.

Only models that support tool calling (function calling) are included in the catalog, since tool use is essential for the agentic workflows that the coding-agent package builds on top of.

## Table of Contents

- [Design Philosophy](#design-philosophy)
- [Core Types](#core-types)
- [Provider Architecture](#provider-architecture)
- [All Supported Providers](#all-supported-providers)
- [Streaming Entry Points](#streaming-entry-points)
- [Event Stream Format](#event-stream-format)
- [ThinkingLevel Abstraction](#thinkinglevel-abstraction)
- [OAuth Support](#oauth-support)
- [Model Catalog](#model-catalog)
- [Cost Tracking](#cost-tracking)
- [Tool Validation](#tool-validation)
- [Provider-Specific Options](#provider-specific-options)
- [Cross-Provider Handoffs](#cross-provider-handoffs)
- [Prompt Caching](#prompt-caching)
- [Adding a New Provider](#adding-a-new-provider)
- [ModelRegistry Breaking Changes](#modelregistry-breaking-changes)
- [Faux Provider](#faux-provider)

---

## Design Philosophy

The library is built around several core principles:

1. **Provider-agnostic context.** The `Context` type (system prompt, messages, tools) is serializable JSON. Conversations can be persisted, resumed, or handed off to a different model without transformation by the caller.

2. **Two-tier API surface.** Provider-native functions (`stream`/`complete`) expose every knob a provider offers. Provider-agnostic functions (`streamSimple`/`completeSimple`) accept a single `reasoning` level and map it to provider-specific parameters automatically.

3. **Lazy loading.** Provider SDK modules are loaded on first use via dynamic `import()`, so importing `@mariozechner/pi-ai` does not pull in the Anthropic, OpenAI, Google, Bedrock, or Mistral SDKs until a request targets that provider.

4. **Browser and Node.** The library works in browsers (API keys passed explicitly) and Node.js (keys resolved from environment variables). Bedrock and OAuth login flows are Node-only.

5. **TypeBox for schemas.** Tool parameter schemas use `@sinclair/typebox`, which produces JSON Schema at runtime and TypeScript types at compile time. The library re-exports `Type`, `Static`, and `TSchema`.

---

## Core Types

All core types are defined in `packages/ai/src/types.ts`.

### Api

The `Api` type identifies a wire protocol. Ten built-in protocols are defined:

```typescript
type KnownApi =
  | "openai-completions"
  | "openai-responses"
  | "openai-codex-responses"
  | "azure-openai-responses"
  | "anthropic-messages"
  | "google-generative-ai"
  | "google-gemini-cli"
  | "google-vertex"
  | "mistral-conversations"
  | "bedrock-converse-stream";

type Api = KnownApi | (string & {});
```

The `(string & {})` union allows custom API identifiers while preserving IDE auto-complete for known values.

### Model

A `Model<TApi>` carries all metadata needed to make a request:

```typescript
interface Model<TApi extends Api> {
  id: string;                    // e.g. "claude-sonnet-4-20250514"
  name: string;                  // human-readable name
  api: TApi;                     // wire protocol
  provider: Provider;            // e.g. "anthropic"
  baseUrl: string;               // API endpoint
  reasoning: boolean;            // supports thinking/reasoning
  input: ("text" | "image")[];   // input modalities
  cost: {
    input: number;               // $/million tokens
    output: number;
    cacheRead: number;
    cacheWrite: number;
  };
  contextWindow: number;         // max input tokens
  maxTokens: number;             // max output tokens
  headers?: Record<string, string>;
  compat?: OpenAICompletionsCompat | OpenAIResponsesCompat;
}
```

The `compat` field is typed conditionally: it is `OpenAICompletionsCompat` for `openai-completions` models, `OpenAIResponsesCompat` for `openai-responses` models, and `never` for other APIs.

### Context

```typescript
interface Context {
  systemPrompt?: string;
  messages: Message[];
  tools?: Tool[];
}
```

`Context` is plain JSON-serializable data. It can be `JSON.stringify`'d, persisted, and `JSON.parse`'d to resume a conversation, including with a different model.

### Message

Three message roles exist:

| Type | `role` | Key Fields |
|------|--------|------------|
| `UserMessage` | `"user"` | `content: string \| (TextContent \| ImageContent)[]`, `timestamp` |
| `AssistantMessage` | `"assistant"` | `content: (TextContent \| ThinkingContent \| ToolCall)[]`, `api`, `provider`, `model`, `usage`, `stopReason`, `responseId?`, `errorMessage?`, `timestamp` |
| `ToolResultMessage` | `"toolResult"` | `toolCallId`, `toolName`, `content: (TextContent \| ImageContent)[]`, `isError`, `details?`, `timestamp` |

Content block types:

- **TextContent** -- `{ type: "text", text: string, textSignature?: string }`
- **ThinkingContent** -- `{ type: "thinking", thinking: string, thinkingSignature?: string, redacted?: boolean }`
- **ImageContent** -- `{ type: "image", data: string, mimeType: string }` (base64-encoded)
- **ToolCall** -- `{ type: "toolCall", id: string, name: string, arguments: Record<string, any>, thoughtSignature?: string }`

### Tool

```typescript
interface Tool<TParameters extends TSchema = TSchema> {
  name: string;
  description: string;
  parameters: TParameters;
}
```

Parameters are TypeBox schemas (which are also valid JSON Schema objects).

### StreamOptions

Base options shared by all providers:

```typescript
interface StreamOptions {
  temperature?: number;
  maxTokens?: number;
  signal?: AbortSignal;
  apiKey?: string;
  transport?: "sse" | "websocket" | "auto";
  cacheRetention?: "none" | "short" | "long";
  sessionId?: string;
  onPayload?: (payload: unknown, model: Model<Api>) => unknown | undefined | Promise<unknown | undefined>;
  headers?: Record<string, string>;
  maxRetryDelayMs?: number;
  metadata?: Record<string, unknown>;
}
```

### SimpleStreamOptions

Extends `StreamOptions` with the provider-agnostic reasoning knob:

```typescript
interface SimpleStreamOptions extends StreamOptions {
  reasoning?: ThinkingLevel;
  thinkingBudgets?: ThinkingBudgets;
}
```

### StopReason

```typescript
type StopReason = "stop" | "length" | "toolUse" | "error" | "aborted";
```

### Usage

```typescript
interface Usage {
  input: number;
  output: number;
  cacheRead: number;
  cacheWrite: number;
  totalTokens: number;
  cost: {
    input: number;
    output: number;
    cacheRead: number;
    cacheWrite: number;
    total: number;
  };
}
```

---

## Provider Architecture

### API Identifiers and Dispatch

The library uses a registry pattern (`src/api-registry.ts`) to map API identifiers to provider implementations. Each registered `ApiProvider` supplies two streaming functions:

```typescript
interface ApiProvider<TApi extends Api, TOptions extends StreamOptions> {
  api: TApi;
  stream: StreamFunction<TApi, TOptions>;
  streamSimple: StreamFunction<TApi, SimpleStreamOptions>;
}
```

When `stream(model, context, options)` is called in `src/stream.ts`, the flow is:

1. Look up the provider via `getApiProvider(model.api)`.
2. Invoke `provider.stream(model, context, options)`.
3. Return the resulting `AssistantMessageEventStream`.

The `complete()` function is a thin wrapper that calls `stream()` and awaits `stream.result()`.

### Lazy Loading

Provider modules are not statically imported. Instead, `src/providers/register-builtins.ts` registers lazy wrappers for each API. On first invocation:

1. A `createLazyStream()` wrapper creates an outer `AssistantMessageEventStream`.
2. The actual provider module is loaded via `import("./anthropic.js")` (or whichever provider).
3. The provider's real stream function is called and its events are forwarded to the outer stream.
4. Subsequent calls reuse the cached module promise.

This design means importing `@mariozechner/pi-ai` does not load any provider SDK. The Anthropic SDK, OpenAI SDK, Google GenAI SDK, etc. are only loaded when a request first targets that API.

### Registration

All ten built-in APIs are registered when `register-builtins.ts` is first imported (which happens at module load via `src/stream.ts`):

```
anthropic-messages        -> anthropic.ts
openai-completions        -> openai-completions.ts
openai-responses          -> openai-responses.ts
openai-codex-responses    -> openai-codex-responses.ts
azure-openai-responses    -> azure-openai-responses.ts
mistral-conversations     -> mistral.ts
google-generative-ai      -> google.ts
google-gemini-cli         -> google-gemini-cli.ts
google-vertex             -> google-vertex.ts
bedrock-converse-stream   -> amazon-bedrock.ts
```

External code can register additional APIs via `registerApiProvider()`, unregister them via `unregisterApiProviders(sourceId)`, or reset to defaults with `resetApiProviders()`.

---

## All Supported Providers

The model catalog (`models.generated.ts`) defines 23 providers. Each maps to one of the 10 wire-protocol APIs:

| Provider | API Identifier | Auth |
|----------|---------------|------|
| `amazon-bedrock` | `bedrock-converse-stream` | AWS credentials (profile, IAM keys, bearer token, ECS roles, IRSA) |
| `anthropic` | `anthropic-messages` | `ANTHROPIC_API_KEY` or `ANTHROPIC_OAUTH_TOKEN` or OAuth |
| `azure-openai-responses` | `azure-openai-responses` | `AZURE_OPENAI_API_KEY` + `AZURE_OPENAI_BASE_URL` or `AZURE_OPENAI_RESOURCE_NAME` |
| `cerebras` | `openai-completions` | `CEREBRAS_API_KEY` |
| `github-copilot` | `anthropic-messages` / `openai-responses` | OAuth (`COPILOT_GITHUB_TOKEN` / `GH_TOKEN` / `GITHUB_TOKEN`) |
| `google` | `google-generative-ai` | `GEMINI_API_KEY` |
| `google-antigravity` | `google-generative-ai` | OAuth |
| `google-gemini-cli` | `google-gemini-cli` | OAuth |
| `google-vertex` | `google-vertex` | `GOOGLE_CLOUD_API_KEY` or ADC |
| `groq` | `openai-completions` | `GROQ_API_KEY` |
| `huggingface` | `openai-completions` | `HF_TOKEN` |
| `kimi-coding` | `anthropic-messages` | `KIMI_API_KEY` |
| `minimax` | `openai-completions` | `MINIMAX_API_KEY` — supported model IDs: `MiniMax-M2.7`, `MiniMax-M2.7-highspeed` (older direct IDs removed in v0.63.0) |
| `minimax-cn` | `openai-completions` | `MINIMAX_CN_API_KEY` — supported model IDs: `MiniMax-M2.7`, `MiniMax-M2.7-highspeed` (older direct IDs removed in v0.63.0) |
| `mistral` | `mistral-conversations` | `MISTRAL_API_KEY` |
| `openai` | `openai-responses` | `OPENAI_API_KEY` |
| `openai-codex` | `openai-codex-responses` | OAuth |
| `opencode` | `openai-completions` | `OPENCODE_API_KEY` |
| `opencode-go` | `openai-completions` | `OPENCODE_API_KEY` |
| `openrouter` | `openai-completions` | `OPENROUTER_API_KEY` |
| `vercel-ai-gateway` | `openai-completions` | `AI_GATEWAY_API_KEY` |
| `xai` | `openai-completions` | `XAI_API_KEY` |
| `zai` | `openai-completions` | `ZAI_API_KEY` |

Additionally, any OpenAI-compatible API (Ollama, vLLM, LM Studio, SGLang) can be used by constructing a custom `Model<"openai-completions">` object.

### Provider Changelog Notes

- **v0.61.0:** Added `gpt-5.4-mini` model for the `openai-codex` provider.
- **v0.62.0:** Added `BedrockOptions.requestMetadata` for AWS cost allocation tagging.
- **v0.63.0 (breaking):** Removed deprecated `minimax` and `minimax-cn` direct model IDs. Use `MiniMax-M2.7` or `MiniMax-M2.7-highspeed`.
- **v0.63.1:** Added `gemini-3.1-pro-preview-customtools` model for the `google-vertex` provider.
- **v0.65.0:** Added tool streaming support for newer Z.ai models.
- **v0.65.0:** Fixed Anthropic HTTP 413 `request_too_large` errors to be detected as context overflow, allowing callers to trigger compaction and retry.

---

## Streaming Entry Points

Defined in `src/stream.ts`, four functions form the public API:

### Provider-Native Pair

```typescript
function stream<TApi extends Api>(
  model: Model<TApi>,
  context: Context,
  options?: ProviderStreamOptions,
): AssistantMessageEventStream;

async function complete<TApi extends Api>(
  model: Model<TApi>,
  context: Context,
  options?: ProviderStreamOptions,
): Promise<AssistantMessage>;
```

`ProviderStreamOptions` is `StreamOptions & Record<string, unknown>`, allowing provider-specific fields (e.g., `thinkingEnabled` for Anthropic, `reasoningEffort` for OpenAI).

### Provider-Agnostic Pair

```typescript
function streamSimple<TApi extends Api>(
  model: Model<TApi>,
  context: Context,
  options?: SimpleStreamOptions,
): AssistantMessageEventStream;

async function completeSimple<TApi extends Api>(
  model: Model<TApi>,
  context: Context,
  options?: SimpleStreamOptions,
): Promise<AssistantMessage>;
```

These accept `SimpleStreamOptions` with a single `reasoning` field of type `ThinkingLevel`. Each provider's `streamSimple` implementation maps this to the appropriate provider-specific parameters.

### When to Use Which

- Use `streamSimple`/`completeSimple` when you want portable code that works across all providers without caring about provider-specific reasoning parameters.
- Use `stream`/`complete` when you need fine-grained control over provider-specific options like `thinkingBudgetTokens` (Anthropic), `reasoningSummary` (OpenAI), or `thinking.budgetTokens` (Google).

---

## Event Stream Format

### AssistantMessageEventStream

`AssistantMessageEventStream` extends `EventStream<AssistantMessageEvent, AssistantMessage>`. It is an `AsyncIterable` that emits events during streaming and resolves a final `AssistantMessage` via `stream.result()`.

The `EventStream` class (`src/utils/event-stream.ts`) uses a push-based queue with async iteration support. Events are either delivered immediately to a waiting consumer or buffered. The stream terminates on a `done` or `error` event.

### Event Types

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

Every event carries a `partial` field with the in-progress `AssistantMessage`. The `contentIndex` identifies which block in `partial.content[]` is being updated.

The `done` event carries the final `AssistantMessage` with complete usage data. The `error` event carries an `AssistantMessage` with `stopReason` set to `"error"` or `"aborted"` and an `errorMessage` field.

### Tool Call Streaming

During `toolcall_delta` events, tool arguments are progressively parsed using the `partial-json` library. The `arguments` field on the in-progress `ToolCall` block always contains at least `{}`, never `undefined`. Fields may be missing, strings may be truncated, and arrays may be incomplete.

The Google provider does not support function call streaming. It emits a single `toolcall_delta` with the full arguments.

---

## ThinkingLevel Abstraction

```typescript
type ThinkingLevel = "minimal" | "low" | "medium" | "high" | "xhigh";
```

This is the provider-agnostic reasoning effort scale used by `streamSimple`/`completeSimple`. Each provider maps it differently.

### Provider Mappings

**Anthropic** (adaptive thinking for Opus 4.6 / Sonnet 4.6):

| ThinkingLevel | Anthropic Effort |
|---------------|-----------------|
| `minimal` | `"low"` |
| `low` | `"low"` |
| `medium` | `"medium"` |
| `high` | `"high"` |
| `xhigh` | `"max"` (Opus 4.6 only, otherwise `"high"`) |

For older Anthropic models (e.g., Sonnet 4), budget-based thinking is used instead of adaptive effort. Default budgets are: minimal=1024, low=2048, medium=8192, high=16384 tokens. These can be overridden via `thinkingBudgets` in `SimpleStreamOptions`.

**OpenAI** (Responses API):

ThinkingLevel maps directly to `reasoningEffort`. `xhigh` is only supported on GPT-5.2/5.3/5.4 models; otherwise it is clamped to `"high"`.

**Google** (Generative AI):

For Gemini 2.5 models, ThinkingLevel maps to token budgets:
- Gemini 2.5 Pro: minimal=128, low=2048, medium=8192, high=32768
- Gemini 2.5 Flash: minimal=128, low=2048, medium=8192, high=24576

For Gemini 3 models, ThinkingLevel maps to Google's native `ThinkingLevel` enum (`MINIMAL`, `LOW`, `MEDIUM`, `HIGH`).

`xhigh` is always clamped to `"high"` on non-OpenAI providers (except Opus 4.6 on Anthropic).

### supportsXhigh()

The `supportsXhigh()` function in `src/models.ts` checks whether a model supports the `xhigh` level:

```typescript
function supportsXhigh<TApi extends Api>(model: Model<TApi>): boolean
```

Returns `true` for GPT-5.2/5.3/5.4 and Opus 4.6 models.

---

## OAuth Support

OAuth authentication is handled by the `@mariozechner/pi-ai/oauth` entry point (defined in `src/utils/oauth/`). The library does not store credentials -- that is the caller's responsibility.

### Supported OAuth Providers

| Provider ID | Name | Mechanism |
|-------------|------|-----------|
| `anthropic` | Anthropic (Claude Pro/Max) | OAuth with bearer token (`sk-ant-oat*`) |
| `openai-codex` | OpenAI Codex (ChatGPT Plus/Pro) | ChatGPT OAuth |
| `github-copilot` | GitHub Copilot | Device code flow |
| `google-gemini-cli` | Google Gemini CLI | Google Cloud OAuth |
| `google-antigravity` | Antigravity | Google Cloud OAuth |

### API Surface

```typescript
// Login functions (each returns OAuthCredentials)
loginAnthropic(callbacks)
loginOpenAICodex(callbacks)
loginGitHubCopilot(callbacks)
loginGeminiCli(callbacks)
loginAntigravity(callbacks)

// Token management
refreshOAuthToken(providerId, credentials)  // -> OAuthCredentials
getOAuthApiKey(providerId, credentialsMap)   // -> { newCredentials, apiKey } | null

// Registry
getOAuthProvider(id)       // -> OAuthProviderInterface
getOAuthProviders()        // -> OAuthProviderInterface[]
registerOAuthProvider(p)   // register custom provider
```

`getOAuthApiKey()` automatically refreshes expired tokens before returning the API key.

### CLI Login

The package includes a CLI binary (`pi-ai`) for interactive OAuth login:

```bash
npx @mariozechner/pi-ai login              # interactive provider selection
npx @mariozechner/pi-ai login anthropic    # login to specific provider
npx @mariozechner/pi-ai list               # list available providers
```

Credentials are saved to `auth.json` in the current directory.

### OAuth Token Detection

The Anthropic provider detects OAuth tokens by checking if the API key contains `"sk-ant-oat"`. When an OAuth token is detected, the request includes Claude Code identity headers (`user-agent: claude-cli/2.1.75`, `anthropic-beta: claude-code-20250219,oauth-2025-04-20,...`). Tool names are also converted to Claude Code canonical casing (e.g., `read` becomes `Read`).

---

## Model Catalog

### Generation

Models are defined in `src/models.generated.ts`, a file auto-generated by `scripts/generate-models.ts`. The build script (`npm run build`) runs the generation step first:

```json
"build": "npm run generate-models && tsgo -p tsconfig.build.json"
```

The generation script fetches model metadata from external sources (e.g., models.dev API) and produces a typed constant `MODELS` that maps providers to their model entries.

### Structure

The `MODELS` object is keyed by provider name, then by model ID. Each entry is a `Model<T>` satisfying the appropriate API type. Example entry:

```typescript
"anthropic": {
  "claude-sonnet-4-20250514": {
    id: "claude-sonnet-4-20250514",
    name: "Claude Sonnet 4",
    api: "anthropic-messages",
    provider: "anthropic",
    baseUrl: "https://api.anthropic.com",
    reasoning: true,
    input: ["text", "image"],
    cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 3.75 },
    contextWindow: 200000,
    maxTokens: 8192,
  } satisfies Model<"anthropic-messages">,
  // ...
}
```

### Registry at Runtime

`src/models.ts` initializes a `Map<string, Map<string, Model<Api>>>` from `MODELS` at module load. Three query functions are exported:

```typescript
getProviders(): KnownProvider[]
getModels(provider): Model[]
getModel(provider, modelId): Model
```

Both `provider` and `modelId` parameters are typed as string literal unions, providing IDE auto-complete for all known provider/model combinations.

### Model Metadata

Each model carries:

- **id** -- the identifier sent to the provider API
- **name** -- human-readable display name
- **api** -- which wire protocol to use
- **provider** -- which provider this model belongs to
- **baseUrl** -- the API endpoint URL
- **reasoning** -- whether the model supports thinking/reasoning
- **input** -- supported input modalities (`"text"` and/or `"image"`)
- **cost** -- pricing per million tokens for input, output, cache read, and cache write
- **contextWindow** -- maximum input context in tokens
- **maxTokens** -- maximum output tokens

---

## Cost Tracking

### calculateCost()

Defined in `src/models.ts`:

```typescript
function calculateCost<TApi extends Api>(model: Model<TApi>, usage: Usage): Usage["cost"]
```

This computes dollar costs from token counts using the model's pricing metadata. The formula for each component is:

```
cost = (model.cost.{component} / 1_000_000) * usage.{component}
```

The total is the sum of input, output, cacheRead, and cacheWrite costs.

### When Costs Are Calculated

Every provider implementation calls `calculateCost(model, output.usage)` after parsing usage metadata from the API response. This happens:

- On `message_start` and `message_delta` events (Anthropic)
- On `response.completed` events (OpenAI Responses)
- On `usageMetadata` chunks (Google)
- On stream completion (Bedrock, Mistral, OpenAI Completions)

The `AssistantMessage` returned by `stream.result()` always has `usage.cost` populated.

### Service Tier Pricing (OpenAI)

The OpenAI Responses provider supports `serviceTier` (`"flex"`, `"priority"`, or default). Cost multipliers are applied after base cost calculation:

- `flex`: 0.5x
- `priority`: 2x
- default: 1x

---

## Tool Validation

### validateToolCall()

Defined in `src/utils/validation.ts`:

```typescript
function validateToolCall(tools: Tool[], toolCall: ToolCall): any
function validateToolArguments(tool: Tool, toolCall: ToolCall): any
```

These validate tool call arguments against the tool's TypeBox schema using AJV. Features:

- **Type coercion** -- AJV is configured with `coerceTypes: true`, so string `"42"` is coerced to number `42` when the schema expects a number.
- **Format validation** -- `ajv-formats` adds support for `date-time`, `email`, `uri`, etc.
- **Structured error messages** -- validation errors include the path, message, and received arguments.
- **Browser extension safety** -- in Chrome extension environments with strict CSP (no `eval`), AJV is disabled and arguments are passed through unvalidated.

The `agentLoop` in the coding-agent package calls `validateToolCall` automatically. When using `stream`/`complete` directly, callers should call it manually after `toolcall_end` events.

### StringEnum Helper

For Google API compatibility, the library provides a `StringEnum` helper (re-exported from `src/utils/typebox-helpers.ts`) instead of `Type.Enum`, which generates `anyOf`/`const` patterns that Google's API does not support.

---

## Provider-Specific Options

Each provider extends `StreamOptions` with its own options interface.

### AnthropicOptions

```typescript
interface AnthropicOptions extends StreamOptions {
  thinkingEnabled?: boolean;
  thinkingBudgetTokens?: number;     // budget-based (older models)
  effort?: AnthropicEffort;          // adaptive (Opus 4.6, Sonnet 4.6)
  interleavedThinking?: boolean;     // default: true
  toolChoice?: "auto" | "any" | "none" | { type: "tool"; name: string };
  client?: Anthropic;                // inject pre-built client
}
```

Adaptive thinking (`type: "adaptive"`) is used for Opus 4.6 and Sonnet 4.6 models. Older models use budget-based thinking (`type: "enabled"` with `budget_tokens`).

### OpenAIResponsesOptions

```typescript
interface OpenAIResponsesOptions extends StreamOptions {
  reasoningEffort?: "minimal" | "low" | "medium" | "high" | "xhigh";
  reasoningSummary?: "auto" | "detailed" | "concise" | null;
  serviceTier?: "flex" | "priority" | "auto" | "default";
}
```

### GoogleOptions

```typescript
interface GoogleOptions extends StreamOptions {
  toolChoice?: "auto" | "none" | "any";
  thinking?: {
    enabled: boolean;
    budgetTokens?: number;            // -1 for dynamic, 0 to disable
    level?: GoogleThinkingLevel;      // "MINIMAL" | "LOW" | "MEDIUM" | "HIGH"
  };
}
```

### BedrockOptions

```typescript
interface BedrockOptions extends StreamOptions {
  requestMetadata?: Record<string, string>;
}
```

#### requestMetadata (v0.62.0+)

Pass key-value pairs for AWS cost allocation tagging. These forward to the Bedrock Converse API `requestMetadata` field and appear in AWS Cost Explorer split cost allocation data:

```typescript
const options: BedrockOptions = {
  requestMetadata: {
    project: "my-app",
    team: "backend",
  },
};
```

`BedrockOptions` is exported from the package root entry point.

### OpenAICompletionsCompat

For OpenAI-compatible providers, the `compat` field on `Model` controls behavior differences:

```typescript
interface OpenAICompletionsCompat {
  supportsStore?: boolean;
  supportsDeveloperRole?: boolean;
  supportsReasoningEffort?: boolean;
  supportsUsageInStreaming?: boolean;
  supportsStrictMode?: boolean;
  maxTokensField?: "max_completion_tokens" | "max_tokens";
  requiresToolResultName?: boolean;
  requiresAssistantAfterToolResult?: boolean;
  requiresThinkingAsText?: boolean;
  thinkingFormat?: "openai" | "openrouter" | "zai" | "qwen" | "qwen-chat-template";
  openRouterRouting?: OpenRouterRouting;
  vercelGatewayRouting?: VercelGatewayRouting;
}
```

If `compat` is not set, the library auto-detects settings from the `baseUrl`. Partial overrides are merged with auto-detected defaults.

---

## Cross-Provider Handoffs

The `transformMessages()` function in `src/providers/transform-messages.ts` enables seamless conversation hand-offs between providers.

When messages from one provider are sent to a different provider:

1. **User and tool result messages** pass through unchanged.
2. **Assistant messages from the same provider/API/model** are preserved as-is, including thinking signatures.
3. **Assistant messages from different providers** have their:
   - Thinking blocks with signatures stripped and converted to plain text blocks (without `<thinking>` tags to avoid the model mimicking them)
   - Redacted thinking blocks dropped entirely (opaque encrypted content is model-specific)
   - `thoughtSignature` on tool calls removed
   - Tool call IDs normalized via a provider-specific function (e.g., Anthropic requires `^[a-zA-Z0-9_-]+$`, max 64 chars)
4. **Errored/aborted assistant messages** are skipped entirely to avoid replaying incomplete turns.
5. **Orphaned tool calls** (tool calls without matching tool results) get synthetic error results inserted.

---

## Prompt Caching

### CacheRetention

```typescript
type CacheRetention = "none" | "short" | "long";
```

Set via `cacheRetention` in `StreamOptions` or the `PI_CACHE_RETENTION` environment variable.

| Provider | `"short"` (default) | `"long"` |
|----------|-------------------|----------|
| Anthropic | 5 minutes (`ephemeral`) | 1 hour (`ephemeral` + `ttl: "1h"`, direct API only) |
| OpenAI Responses | in-memory | 24 hours (`prompt_cache_retention: "24h"`, direct API only) |

The `sessionId` field in `StreamOptions` enables session-based caching for providers that support it (OpenAI Codex uses it for `prompt_cache_key`).

---

## Adding a New Provider

Based on the project's AGENTS.md and the existing codebase, adding a provider requires changes across 8 areas:

### 1. Core Types (`src/types.ts`)

- Add the API identifier to `KnownApi` (e.g., `"my-new-api"`)
- Create an options interface extending `StreamOptions` (e.g., `MyNewApiOptions`)
- Add the provider name to `KnownProvider` (e.g., `"my-provider"`)

### 2. Provider Implementation (`src/providers/`)

Create a new file (e.g., `my-provider.ts`) that exports:

- `streamMyProvider()` -- a `StreamFunction` returning `AssistantMessageEventStream`
- `streamSimpleMyProvider()` -- maps `SimpleStreamOptions` to provider-specific options
- Message conversion functions
- Tool conversion functions
- Response parsing to emit standardized events

### 3. API Registry (`src/providers/register-builtins.ts`)

- Add lazy loader wrappers (never statically import provider modules here)
- Register via `registerApiProvider({ api: "my-new-api", stream: ..., streamSimple: ... })`

### 4. Package Exports (`package.json`)

Add a subpath export:

```json
"./my-provider": {
  "types": "./dist/providers/my-provider.d.ts",
  "import": "./dist/providers/my-provider.js"
}
```

### 5. Public Re-exports (`src/index.ts`)

Add `export type { MyNewApiOptions } from "./providers/my-provider.js";`

### 6. Credential Detection (`src/env-api-keys.ts`)

Add the provider's environment variable to the `envMap` or add custom logic for complex auth.

### 7. Model Generation (`scripts/generate-models.ts`)

Add logic to fetch models from the provider's source and map to the `Model` interface.

### 8. Tests (`test/`)

Add the provider to:
- `stream.test.ts` -- basic streaming and tool use
- `tokens.test.ts` -- token usage reporting
- `abort.test.ts` -- request cancellation
- `cross-provider-handoff.test.ts` -- at least one model pair
- Other test files as applicable

### 9. Documentation

- Update `packages/ai/README.md` supported providers list and environment variables table
- Add entry to `packages/ai/CHANGELOG.md`
- Update `../coding-agent/src/core/model-resolver.ts` with default model
- Update `../coding-agent/src/cli/args.ts` help text

---

## Environment Variable Detection

`src/env-api-keys.ts` provides `getEnvApiKey(provider)` for resolving API keys from environment variables in Node.js. The function:

- Returns `undefined` in browser environments (dynamic import guards prevent loading `node:fs`, `node:os`, `node:path`)
- Handles special cases:
  - **GitHub Copilot**: checks `COPILOT_GITHUB_TOKEN`, then `GH_TOKEN`, then `GITHUB_TOKEN`
  - **Anthropic**: `ANTHROPIC_OAUTH_TOKEN` takes precedence over `ANTHROPIC_API_KEY`
  - **Google Vertex**: checks `GOOGLE_CLOUD_API_KEY`, then falls back to ADC file detection + project/location env vars
  - **Amazon Bedrock**: checks 6 different AWS credential sources (profile, IAM keys, bearer token, ECS roles, IRSA)
- All other providers use a simple key-value map (e.g., `openai` -> `OPENAI_API_KEY`)

---

## ModelRegistry Breaking Changes

`ModelRegistry` is defined in `@mariozechner/pi-agent-core` (the coding-agent package), not in `@mariozechner/pi-ai` itself. Consumers who use `ModelRegistry` for API key resolution should note these changes.

**v0.63.0:** `ModelRegistry.getApiKey(model)` was removed. Use `getApiKeyAndHeaders(model)` instead, which returns `{ apiKey, headers }`:

```typescript
// Before (removed in v0.63.0)
const apiKey = await ctx.modelRegistry.getApiKey(model);

// After
const { apiKey, headers } = await ctx.modelRegistry.getApiKeyAndHeaders(model);
```

**v0.64.0:** `ModelRegistry` no longer has a public constructor. Use factory methods:

```typescript
// File-backed registry (reads ~/.pi/models.json)
const registry = ModelRegistry.create(authStorage);
const registry = ModelRegistry.create(authStorage, customModelsJsonPath);

// In-memory/test registry (built-in models only, no file I/O)
const registry = ModelRegistry.inMemory(authStorage);

// Direct `new ModelRegistry(...)` no longer compiles.
```

---

## Faux Provider

### Faux Provider (v0.64.0+)

For deterministic tests and scripted demos, register a faux provider that returns scripted responses:

```typescript
import {
  registerFauxProvider,
  fauxAssistantMessage,
  fauxText,
  fauxThinking,
  fauxToolCall,
} from "@mariozechner/pi-ai";

registerFauxProvider();
// Use provider: "faux" when constructing a model
```

All faux helpers are exported from the package root via `export * from "./providers/faux.js"`. Use `fauxText`, `fauxThinking`, and `fauxToolCall` to build scripted `AssistantMessage` content, and `fauxAssistantMessage` to assemble full messages.

---

## Package Metadata

- **Package name**: `@mariozechner/pi-ai`
- **Version**: 0.65.2
- **License**: MIT
- **Author**: Mario Zechner
- **Repository**: `github.com/badlogic/pi-mono` (directory: `packages/ai`)
- **Node requirement**: >= 20.0.0
- **Module format**: ESM only (`"type": "module"`)
- **Key dependencies**: `@anthropic-ai/sdk`, `openai`, `@google/genai`, `@mistralai/mistralai`, `@aws-sdk/client-bedrock-runtime`, `@sinclair/typebox`, `ajv`, `partial-json`
