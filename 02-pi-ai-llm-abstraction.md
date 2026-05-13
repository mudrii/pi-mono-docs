# pi-ai: Unified LLM Abstraction

**Package:** `@earendil-works/pi-ai`
**Version:** 0.74.0
**Source:** `packages/ai/` in `earendil-works/pi-mono` @ tag `v0.74.0` (commit `1eee081e`)
**Role:** Provider-agnostic streaming LLM client used by `@earendil-works/pi-tui`, `@earendil-works/pi-cli`, and downstream agent runtimes.

---

## 1. What This Package Does

`pi-ai` exposes a single streaming API (`stream`, `complete`, `streamSimple`, `completeSimple`) that drives 31 model providers across 9 wire protocols. A typed `Model` descriptor selects the provider; a registered `ApiProvider` translates the unified `Message[]` history into provider-native requests, normalizes the response stream into an `AssistantMessageEventStream`, and reports cached/uncached token usage with per-million pricing.

Key design points:

1. **One message shape, many wires.** `UserMessage | AssistantMessage | ToolResultMessage` (see `packages/ai/src/types.ts:140-233`) is consumed identically by Anthropic Messages, OpenAI Chat Completions, OpenAI Responses, Google GenAI, Vertex, Bedrock Converse, Mistral, Azure, and Codex.
2. **Lazy provider loading.** Built-in providers register via dynamic `import()` wrappers (`packages/ai/src/providers/register-builtins.ts:1-403`) — SDKs are only loaded when a stream call hits that API.
3. **Cross-provider continuity.** `transformMessages` (`packages/ai/src/providers/transform-messages.ts:1-220`) rewrites assistant history when handing off between providers so thinking blocks, tool-call IDs, and image content are valid for the receiving model.
4. **Pluggable.** `registerApiProvider`, `registerFauxProvider`, `registerOAuthProvider`, and `MODELS` extension hooks let downstream code add wire protocols and custom catalogs without forking.

---

## 2. Public API Surface

### 2.1 Package exports

`packages/ai/package.json` declares the following subpath exports for v0.74.0:

| Subpath | Purpose |
|---|---|
| `.` | Root entry — re-exports types, `stream`/`complete`, registry, models, OAuth (`packages/ai/src/index.ts:1-43`) |
| `./anthropic` | `AnthropicOptions`, Anthropic provider module |
| `./azure-openai-responses` | Azure-flavoured OpenAI Responses |
| `./google` | Google GenAI (Gemini) provider |
| `./google-vertex` | Vertex AI provider |
| `./mistral` | Mistral conversations API |
| `./openai-codex-responses` | ChatGPT Codex (WebSocket) provider |
| `./openai-completions` | OpenAI-compatible chat completions (covers xAI, Groq, DeepSeek, etc.) |
| `./openai-responses` | OpenAI Responses API |
| `./oauth` | OAuth registry + built-in providers |
| `./bedrock-provider` | Amazon Bedrock Converse Stream |

Binary: `pi-ai` → `dist/cli.js` (interactive REPL + provider login + listing).

### 2.2 Top-level functions

From `packages/ai/src/index.ts:1-43`:

```ts
// Streaming
import { stream, complete, streamSimple, completeSimple } from "@earendil-works/pi-ai";

// Catalog
import { getModel, getProviders, getModels, calculateCost } from "@earendil-works/pi-ai";

// Registry
import { registerApiProvider, unregisterApiProviders, getApiProvider } from "@earendil-works/pi-ai";

// OAuth
import { getOAuthProvider, registerOAuthProvider, getOAuthApiKey } from "@earendil-works/pi-ai";

// Faux (testing)
import { registerFauxProvider, fauxText, fauxThinking, fauxToolCall } from "@earendil-works/pi-ai";

// Utilities
import { isContextOverflow } from "@earendil-works/pi-ai";
```

### 2.3 Core types

All defined in `packages/ai/src/types.ts`:

| Type | Location | Purpose |
|---|---|---|
| `KnownApi` | `types.ts:6-15` | Union of 9 built-in wire protocols |
| `KnownProvider` | `types.ts:19-50` | Union of 31 built-in provider IDs |
| `Model<TApi>` | `types.ts:434-464` | Generic model descriptor; conditional `compat` payload narrows by API |
| `Message` | `types.ts:140-158` | `UserMessage \| AssistantMessage \| ToolResultMessage` |
| `AssistantMessage` | `types.ts:220-233` | `content`, `model`, `provider`, `api`, `usage`, `stopReason`, `errorMessage?` |
| `Usage` | `types.ts:197-210` | `input`, `output`, `cacheRead`, `cacheWrite`, `cost{input,output,cacheRead,cacheWrite,total}` |
| `StopReason` | `types.ts:212-218` | `"stop" \| "length" \| "toolUse" \| "error" \| "aborted"` |
| `AssistantMessageEvent` | `types.ts:269-281` | 13 event types streamed during generation |
| `StreamOptions` | `types.ts:75-136` | Cancellation, telemetry, retries, headers, transport, session caching |
| `ThinkingLevel` | `types.ts:60-66` | `"minimal" \| "low" \| "medium" \| "high" \| "xhigh"` |
| `ModelThinkingLevel` | — | Adds `"off"` for models that can toggle reasoning |
| `OpenAICompletionsCompat` | `types.ts:287-322` | Per-model compatibility flags (max-tokens field, thinking format, cache control format) |
| `OpenRouterRouting` | `types.ts:352-419` | OpenRouter routing/preset/transforms/provider preferences |
| `VercelGatewayRouting` | `types.ts:426-431` | Vercel AI Gateway routing |

---

## 3. Streaming Contract

### 3.1 Entry points

`packages/ai/src/stream.ts:1-59` imports `./providers/register-builtins.js` for side-effect registration, then dispatches via `getApiProvider(model.api)`:

```ts
function stream(args: StreamArgs): AssistantMessageEventStream;
function complete(args: StreamArgs): Promise<AssistantMessage>;
function streamSimple(args: SimpleStreamArgs): AssistantMessageEventStream;
function completeSimple(args: SimpleStreamArgs): Promise<AssistantMessage>;
```

`StreamArgs` (canonical form):

```ts
type StreamArgs = {
    model: Model;
    messages: Message[];
    apiKey: string;
    systemPrompt?: string;
    tools?: Tool[];
    options?: StreamOptions & ProviderSpecificOptions;
};
```

The `*Simple` variants accept a flat string prompt and return single-turn output.

### 3.2 Event stream

`packages/ai/src/utils/event-stream.ts:1-88` defines `EventStream<T, R>` with internal `queue`/`waiting`/`done`/`finalResultPromise`. `AssistantMessageEventStream` resolves the final `AssistantMessage` when an event of type `"done"` or `"error"` is pushed.

Event types (`types.ts:269-281`):

| Event | When |
|---|---|
| `start` | Stream opened |
| `text_start` / `text_delta` / `text_end` | Plain assistant text |
| `thinking_start` / `thinking_delta` / `thinking_end` | Reasoning content (when model supports it) |
| `toolcall_start` / `toolcall_delta` / `toolcall_end` | Tool call streaming |
| `done` | Stream finished cleanly |
| `error` | Stream aborted with `errorMessage` |

Consumer pattern:

```ts
const ev = stream({ model, messages, apiKey });
for await (const event of ev) {
    if (event.type === "text_delta") process.stdout.write(event.delta);
}
const msg = await ev.finalResult; // AssistantMessage
```

### 3.3 StreamOptions

From `types.ts:75-136`:

| Field | Purpose |
|---|---|
| `signal?: AbortSignal` | Cancellation |
| `onPayload?(payload)` | Inspect outgoing wire payload |
| `onResponse?(response)` | Inspect raw HTTP response |
| `timeoutMs?: number` | Per-request timeout |
| `maxRetries?: number` | Provider retry budget |
| `maxRetryDelayMs?: number` | Exponential backoff cap |
| `metadata?: Record<string,string>` | Forwarded to provider (Bedrock requestMetadata, Anthropic metadata) |
| `headers?: Record<string,string>` | Additional HTTP headers |
| `sessionId?: string` | Reuse WebSocket session (Codex) |
| `transport?: "sse" \| "websocket" \| "websocket-cached" \| "auto"` | Wire selection (Codex) |
| `cacheRetention?: "none" \| "short" \| "long"` | Anthropic ephemeral cache TTL |

---

## 4. Provider Catalog

Built-in providers (`KnownProvider`, `types.ts:19-50`) grouped by API family. All sourced from `packages/ai/src/models.generated.ts` — line offsets below mark the section in that file.

### 4.1 Anthropic Messages API (`anthropic-messages`)

| Provider | `models.generated.ts` offset | Notes |
|---|---|---|
| `anthropic` | 1599 | Direct Anthropic SDK; supports `client` override (e.g. `AnthropicVertex`) |

Options (`packages/ai/src/providers/anthropic.ts:174-217`):

- `thinkingEnabled?: boolean`
- `thinkingBudgetTokens?: number`
- `effort?: "low" \| "medium" \| "high" \| "xhigh" \| "max"`
- `thinkingDisplay?: "summarized" \| "omitted"` (default `"summarized"`)
- `interleavedThinking?: boolean` — sets `anthropic-beta: interleaved-thinking-2025-05-14`
- `toolChoice?: ...`
- `client?: AnthropicSDK` — pre-built SDK instance for proxying or Vertex routing

OAuth (Claude Code): the provider attaches `anthropic-beta: oauth-2025-04-20, fine-grained-tool-streaming-2025-05-14, interleaved-thinking-2025-05-14` and rewrites the system prompt when `apiKey` comes from `getOAuthApiKey("anthropic")`.

### 4.2 OpenAI Chat Completions (`openai-completions`)

| Provider | offset | Compat notes |
|---|---|---|
| `cerebras` | 2734 | `isNonStandard: true`, `supportsStore: false`, `supportsDeveloperRole: false` |
| `cloudflare-ai-gateway` | 2804 | `maxTokensField: "max_tokens"` |
| `cloudflare-workers-ai` | 3414 | `isNonStandard: true` |
| `deepseek` | 3560 | `thinkingFormat: "deepseek"` |
| `fireworks` | 3600 | Standard compat |
| `github-copilot` | 3925 | OAuth (device flow), per-request `X-GitHub-Token` header |
| `groq` | 5114 | Standard |
| `huggingface` | 5423 | Standard |
| `kimi-coding` | 5821 | Moonshot AI router; `thinkingFormat: "qwen"` for K2 P-series |
| `minimax` | 5859 | Standard |
| `minimax-cn` | 5895 | CN endpoint |
| `moonshotai` | 6409 | `maxTokensField: "max_tokens"` |
| `moonshotai-cn` | 6537 | CN endpoint |
| `opencode` | 7585 | `isNonStandard: true` |
| `opencode-go` | 8253 | Go runtime backend |
| `openrouter` | 8465 | `OpenRouterRouting` payload via `compat.openrouter` (`types.ts:352-419`) |
| `vercel-ai-gateway` | 13161 | `VercelGatewayRouting` |
| `xai` | 15931 | `isNonStandard: true` for grok-4 |
| `xiaomi` | 16358 | Direct `api.xiaomimimo.com` endpoint (v0.73.0 BREAKING change) |
| `xiaomi-token-plan-ams` | 16445 | AMS region |
| `xiaomi-token-plan-cn` | 16532 | CN region |
| `xiaomi-token-plan-sgp` | 16619 | SGP region |
| `zai` | 16706 | `thinkingFormat: "zai"`; silent context overflow handled in `utils/overflow.ts` |

`detectCompat` in `packages/ai/src/providers/openai-completions.ts:1048-1102` infers compat from `model.provider` and `baseUrl`:

- `thinkingFormat`: `"deepseek"` for DeepSeek, `"zai"` for Z.ai, `"openrouter"` for OpenRouter, `"qwen"` / `"qwen-chat-template"` for Qwen variants, `"openai"` otherwise.
- `cacheControlFormat: "anthropic"` only when `provider === "openrouter"` AND `model.id.startsWith("anthropic/")`.
- `maxTokensField`: `"max_tokens"` for Chutes / Moonshot / Cloudflare AI Gateway; `"max_completion_tokens"` elsewhere.

Options (`packages/ai/src/providers/openai-completions.ts:77-80`):
- `toolChoice?: "auto" \| "none" \| "required" \| { name: string }`
- `reasoningEffort?: "minimal" \| "low" \| "medium" \| "high" \| "xhigh"`

### 4.3 OpenAI Responses (`openai-responses`)

| Provider | offset |
|---|---|
| `openai` | 6665 |

Options (`packages/ai/src/providers/openai-responses.ts:55-59`):
- `reasoningEffort?: "minimal" \| "low" \| "medium" \| "high" \| "xhigh"`
- `reasoningSummary?: "auto" \| "detailed" \| "concise" \| null`
- `serviceTier?: "auto" \| "default" \| "flex" \| "priority" \| "scale"`

### 4.4 Azure OpenAI Responses (`azure-openai-responses`)

| Provider | offset |
|---|---|
| `azure-openai-responses` | 1994 |

Options (`packages/ai/src/providers/azure-openai-responses.ts:44-51`) extend Responses options with:
- `azureApiVersion?: string` (default `"v1"`)
- `azureResourceName?: string`
- `azureBaseUrl?: string`
- `azureDeploymentName?: string`

`AZURE_OPENAI_DEPLOYMENT_NAME_MAP` env var is parsed as comma-separated `model-id=deployment-name` pairs to override per-model deployment routing.

### 4.5 OpenAI Codex Responses (`openai-codex-responses`)

| Provider | offset |
|---|---|
| `openai-codex` | 7405 |

`DEFAULT_CODEX_BASE_URL = "https://chatgpt.com/backend-api"` (`packages/ai/src/providers/openai-codex-responses.ts:50`).

Options (`openai-codex-responses.ts:70-75`):
- `reasoningEffort?: "none" \| "minimal" \| "low" \| "medium" \| "high" \| "xhigh"`
- `reasoningSummary?: "auto" \| "concise" \| "detailed" \| "off" \| "on" \| null`
- `serviceTier?` (same as OpenAI Responses)
- `textVerbosity?: "low" \| "medium" \| "high"`

Transport selection (line 184): when `options.transport === "websocket"` or `"websocket-cached"`, the provider establishes a WebSocket session keyed by `sessionId`. Cached sessions expire after 5 minutes of inactivity. Debug helpers exported:

```ts
import {
    getOpenAICodexWebSocketDebugStats,
    resetOpenAICodexWebSocketDebugStats,
    closeOpenAICodexWebSocketSessions,
} from "@earendil-works/pi-ai/openai-codex-responses";
```

### 4.6 Google Generative AI (`google-generative-ai`)

| Provider | offset |
|---|---|
| `google` | 4419 |

Options (`packages/ai/src/providers/google.ts:36-43`):
- `toolChoice?: "auto" \| "none" \| "any"`
- `thinking?: { enabled: boolean; budgetTokens?: number; level?: GoogleThinkingLevel }`
  - `budgetTokens = -1` → dynamic, `0` → disabled.

### 4.7 Google Vertex (`google-vertex`)

| Provider | offset |
|---|---|
| `google-vertex` | 4887 |

Options (`packages/ai/src/providers/google-vertex.ts:38-47`) extend `GoogleOptions` with `project` and `location`. Authenticates with API key OR Application Default Credentials (ADC): detected via `GOOGLE_APPLICATION_CREDENTIALS` or `~/.config/gcloud/application_default_credentials.json` (`env-api-keys.ts:63-89`).

### 4.8 Mistral Conversations (`mistral-conversations`)

| Provider | offset |
|---|---|
| `mistral` | 5931 |

Options (`packages/ai/src/providers/mistral.ts:40-44`):
- `toolChoice?: "auto" \| "none" \| "any"`
- `promptMode?: "reasoning"`
- `reasoningEffort?: "none" \| "high"`

Each request constructs a fresh `MistralClient`; no shared mutable state.

### 4.9 Amazon Bedrock Converse Stream (`bedrock-converse-stream`)

| Provider | offset |
|---|---|
| `amazon-bedrock` | 7 |

Options (`packages/ai/src/providers/amazon-bedrock.ts:49-83`):
- `region?: string`
- `profile?: string`
- `toolChoice?`
- `reasoning?: boolean`
- `thinkingBudgets?: number`
- `interleavedThinking?: boolean` — Claude 4.x only
- `thinkingDisplay?: "summarized" \| "omitted"`
- `requestMetadata?: Record<string,string>` — ≤50 pairs, keys ≤64 chars (no `aws:` prefix), values ≤256 chars
- `bearerToken?: string` — when set, sends `Authorization: Bearer <token>` and bypasses SigV4

Auth chain (`env-api-keys.ts:182-205`): `AWS_PROFILE` → `AWS_ACCESS_KEY_ID + AWS_SECRET_ACCESS_KEY` → `AWS_BEARER_TOKEN_BEDROCK` → container creds → IRSA.

The Bedrock provider is loaded node-only and supports `setBedrockProviderModule()` override for non-node runtimes (`register-builtins.ts`).

---

## 5. Model Resolution

`packages/ai/src/models.ts:20-37` defines:

```ts
function getModel<TProvider extends KnownProvider, TId extends keyof MODELS[TProvider]>(
    provider: TProvider,
    id: TId,
): MODELS[TProvider][TId];

function getProviders(): KnownProvider[];
function getModels(provider: KnownProvider): Model[];
```

The `MODELS` interface is augmented by `models.generated.ts` for every provider; downstream packages can declare-merge their own provider catalogs.

### 5.1 Cost calculation

`models.ts:39-46`:

```ts
function calculateCost(model: Model, usage: Usage): Usage["cost"] {
    return {
        input:     (model.cost.input      / 1_000_000) * usage.input,
        output:    (model.cost.output     / 1_000_000) * usage.output,
        cacheRead: (model.cost.cacheRead  / 1_000_000) * usage.cacheRead,
        cacheWrite:(model.cost.cacheWrite / 1_000_000) * usage.cacheWrite,
        total: /* sum of the above */,
    };
}
```

Costs in `models.generated.ts` are quoted **per 1M tokens**. The streaming pipeline calls `calculateCost` automatically and attaches the result to `AssistantMessage.usage.cost`.

### 5.2 Thinking level clamping

`models.ts:50-80`:

- `getSupportedThinkingLevels(model)` reads `model.reasoning?.thinkingLevels` or returns the canonical `["minimal","low","medium","high","xhigh"]`.
- `clampThinkingLevel(model, level)` walks `EXTENDED_THINKING_LEVELS` and selects the nearest supported level. Models with `thinkingLevelMap` (e.g. Bedrock Claude variants) use the map's keys as the supported set.

---

## 6. Cross-Provider Message Transformation

When a conversation begun on one model is continued on another, `transformMessages` (`packages/ai/src/providers/transform-messages.ts:1-220`) rewrites the history in two passes.

### 6.1 Pass 1 — content rewrites

For each `AssistantMessage` where `(model.id, model.provider, model.api)` differs from the incoming target:

- **Thinking blocks** are downgraded to plain text. The receiving provider would otherwise reject signed reasoning blocks (`thoughtSignature` mismatch).
- **Tool-call `thoughtSignature`** is stripped — the signature is provider-specific (e.g. Anthropic).
- **Image blocks** are replaced with the placeholder `[image omitted]` when the target model is not vision-capable (`downgradeUnsupportedImages`).
- **Tool-call IDs** are normalized via the optional `normalizeToolCallId(id)` callback. Mistral truncates to 9 chars, OpenAI keeps original IDs.

Messages with `stopReason === "error"` or `"aborted"` are skipped entirely so partial state doesn't leak across providers.

### 6.2 Pass 2 — orphan tool-call patching

If an assistant message ends with `toolcall_*` blocks but no subsequent `ToolResultMessage` exists, a synthetic message is inserted:

```ts
{ role: "toolResult", content: "No result provided", isError: true, toolCallId: ... }
```

This satisfies providers (Anthropic, OpenAI) that reject conversations with unbalanced tool calls.

### 6.3 When transformation runs

The dispatch path is: `stream()` → provider entry → `transformMessages({ messages, model, normalizeToolCallId? })`. Providers that need ID normalization (Mistral) pass their callback; others rely on defaults.

---

## 7. Authentication

### 7.1 Environment variables

`packages/ai/src/env-api-keys.ts:101-129` maps each provider to one or more env vars. Highlights:

| Provider | Env vars (priority) |
|---|---|
| `anthropic` | `ANTHROPIC_OAUTH_TOKEN` > `ANTHROPIC_API_KEY` |
| `openai` / `openai-responses` | `OPENAI_API_KEY` |
| `azure-openai-responses` | `AZURE_OPENAI_API_KEY` (+ `AZURE_OPENAI_DEPLOYMENT_NAME_MAP`) |
| `openai-codex` | `OPENAI_CODEX_OAUTH_TOKEN` (refreshed via OAuth) |
| `google` | `GEMINI_API_KEY` |
| `google-vertex` | API key OR ADC |
| `github-copilot` | `COPILOT_GITHUB_TOKEN` > `GH_TOKEN` > `GITHUB_TOKEN` |
| `amazon-bedrock` | AWS chain (see §4.9) |
| `groq` | `GROQ_API_KEY` |
| `xai` | `XAI_API_KEY` |
| `deepseek` | `DEEPSEEK_API_KEY` |
| `mistral` | `MISTRAL_API_KEY` |
| `cerebras` | `CEREBRAS_API_KEY` |
| `openrouter` | `OPENROUTER_API_KEY` |
| `fireworks` | `FIREWORKS_API_KEY` |
| `cloudflare-workers-ai` | `CLOUDFLARE_AI_API_KEY` |
| `cloudflare-ai-gateway` | `CLOUDFLARE_AI_GATEWAY_API_KEY` |
| `zai` | `ZAI_API_KEY` |
| `moonshotai` / `kimi-coding` | `MOONSHOT_API_KEY` / `KIMI_API_KEY` |
| `xiaomi*` | `XIAOMI_API_KEY` |

On Bun, when `process.env` is empty, `env-api-keys.ts` falls back to reading `/proc/self/environ`.

### 7.2 OAuth

`packages/ai/src/utils/oauth/index.ts:1-153`.

Built-in OAuth providers (`BUILT_IN_OAUTH_PROVIDERS`): `anthropic`, `github-copilot`, `openai-codex`.

Interface (`utils/oauth/types.ts:1-72`):

```ts
type OAuthCredentials = { refresh: string; access: string; expires: number };

type OAuthProviderInterface = {
    id: string;
    name: string;
    login(callbacks: OAuthLoginCallbacks): Promise<OAuthCredentials>;
    refreshToken(refresh: string): Promise<OAuthCredentials>;
    getApiKey(creds: OAuthCredentials): Promise<string>;
    modifyModels?(models: Model[]): Model[];
    usesCallbackServer?: boolean;
};
```

`getOAuthApiKey(providerId, creds)` auto-refreshes if `Date.now() >= creds.expires`.

| Provider | Flow | Callback URL |
|---|---|---|
| `anthropic` | PKCE | `http://localhost:53692/oauth/callback` (`utils/oauth/anthropic.ts:31-34`) |
| `openai-codex` | PKCE | `http://localhost:1455/auth/callback` (`utils/oauth/openai-codex.ts:24-28`) |
| `github-copilot` | Device flow | n/a — polls GitHub device-code endpoint |

Custom OAuth providers: `registerOAuthProvider({ id, name, login, refreshToken, getApiKey })`. `resetOAuthProviders()` restores the built-in set (useful for tests).

---

## 8. Overflow Detection

`packages/ai/src/utils/overflow.ts:1-153` exposes `isContextOverflow(message, contextWindow?)` that returns `true` when an `AssistantMessage` represents a context-window overflow. It handles three cases:

1. **Error-text patterns** — 20 regex patterns covering Anthropic (`prompt is too long`, `request_too_large`), OpenAI (`exceeds the context window`), Google (`input token count … exceeds the maximum`), xAI (`maximum prompt length is N`), Groq, OpenRouter, llama.cpp, LM Studio, MiniMax, Kimi for Coding, Mistral, Cerebras (`400/413 (no body)`), Ollama, and generic fallbacks (`context_length_exceeded`, `too many tokens`, `token limit exceeded`). Non-overflow patterns (`rate limit`, `too many requests`, Bedrock throttling) are excluded.
2. **Silent overflow (z.ai)** — `stopReason === "stop"` but `usage.input + usage.cacheRead > contextWindow`.
3. **Length-stop overflow (Xiaomi MiMo)** — `stopReason === "length"` AND `usage.output === 0` AND `usage.input + usage.cacheRead >= contextWindow * 0.99` (server truncates input to fit, no room left to generate).

---

## 9. Registry & Extensibility

### 9.1 API provider registry

`packages/ai/src/api-registry.ts:1-98` keeps a `Map<string, RegisteredApiProvider>`. `registerApiProvider` wraps `stream` and `streamSimple` with an API-mismatch guard that throws `Mismatched api: X expected Y` if a caller hands a model whose `api` doesn't match the registered one. `unregisterApiProviders(sourceId)` removes all providers tagged with a given source ID — used by extension hosts to unload plugin SDKs cleanly.

### 9.2 Built-in registration

`packages/ai/src/providers/register-builtins.ts:1-403` defines `createLazyStream`/`createLazySimpleStream` wrappers that dynamically `import()` the provider module on first call. `registerBuiltInApiProviders()` is invoked at module load (line 403) and registers all 9 APIs. Bedrock has a node-only import path and a `setBedrockProviderModule()` override for non-node runtimes (Bun, Deno, edge).

### 9.3 Faux provider (testing)

`packages/ai/src/providers/faux.ts:37-126` lets tests script deterministic streams:

```ts
import { registerFauxProvider, fauxText, fauxThinking, fauxToolCall } from "@earendil-works/pi-ai";

const faux = registerFauxProvider({
    api: "openai-completions",
    provider: "faux",
    models: [{ id: "test-model", cost: { input: 1, output: 2, cacheRead: 0, cacheWrite: 0 }, contextWindow: 200_000, maxTokens: 8192 }],
    tokensPerSecond: 50,
    tokenSize: 4,
});

faux.setResponses([
    [fauxText("Hello "), fauxText("world")],
    [fauxThinking("planning..."), fauxToolCall("search", { q: "x" })],
]);

// stream({ model: faux.getModel("test-model"), ... }) emits scripted events.
faux.unregister();
```

`faux.getPendingResponseCount()` and `faux.appendResponses(...)` enable iterative test scenarios.

---

## 10. Examples

### 10.1 Single-turn streaming (Anthropic)

```ts
import { stream, getModel } from "@earendil-works/pi-ai";

const model = getModel("anthropic", "claude-sonnet-4-5");
const ev = stream({
    model,
    apiKey: process.env.ANTHROPIC_API_KEY!,
    messages: [{ role: "user", content: "Summarize Conway's Law in one sentence." }],
});

for await (const e of ev) {
    if (e.type === "text_delta") process.stdout.write(e.delta);
}
const final = await ev.finalResult;
console.log("\ncost:", final.usage.cost.total);
```

### 10.2 Tool calls with reasoning (OpenAI Responses)

```ts
import { stream, getModel } from "@earendil-works/pi-ai";
import type { Tool } from "@earendil-works/pi-ai";

const model = getModel("openai", "gpt-5.1");
const tools: Tool[] = [{
    name: "get_weather",
    description: "Get the current weather",
    inputSchema: { type: "object", properties: { city: { type: "string" } }, required: ["city"] },
}];

const ev = stream({
    model,
    apiKey: process.env.OPENAI_API_KEY!,
    messages: [{ role: "user", content: "What's the weather in SF?" }],
    tools,
    options: { reasoningEffort: "medium", reasoningSummary: "concise" },
});

const result = await ev.finalResult;
if (result.stopReason === "toolUse") {
    // Execute the tool call, then continue the conversation
    // by appending a toolResult message.
}
```

### 10.3 Cross-provider handoff

```ts
import { stream, getModel } from "@earendil-works/pi-ai";

let messages = [{ role: "user" as const, content: "Draft a sonnet about TCP." }];

// Turn 1 with Anthropic
const claude = getModel("anthropic", "claude-opus-4-7");
const r1 = await stream({ model: claude, apiKey: anthropicKey, messages }).finalResult;
messages = [...messages, r1];

// Turn 2 hands off to OpenAI — transformMessages downgrades Claude's thinking blocks
const gpt = getModel("openai", "gpt-5.1");
const r2 = await stream({
    model: gpt,
    apiKey: openaiKey,
    messages: [...messages, { role: "user", content: "Now critique your own poem." }],
}).finalResult;
```

### 10.4 OAuth login (Claude Code)

```ts
import { getOAuthProvider, getOAuthApiKey } from "@earendil-works/pi-ai";

const anthropic = getOAuthProvider("anthropic")!;
const creds = await anthropic.login({
    onPrompt: (url) => console.log(`Open ${url} in your browser`),
    onAuth:   (c)   => persistToDisk(c),
});

// Later — auto-refresh if expired
const apiKey = await getOAuthApiKey("anthropic", creds);
```

### 10.5 Overflow guard + retry on smaller model

```ts
import { complete, getModel, isContextOverflow } from "@earendil-works/pi-ai";

async function safeComplete(messages: Message[]) {
    const primary = getModel("anthropic", "claude-opus-4-7");
    const result = await complete({ model: primary, apiKey: key, messages });

    if (isContextOverflow(result, primary.contextWindow)) {
        const fallback = getModel("google", "gemini-2.5-pro");
        return complete({ model: fallback, apiKey: geminiKey, messages });
    }
    return result;
}
```

---

## 11. Session Resources

`packages/ai/src/session-resources.ts` (exported from `index.ts:1-43`):

```ts
import { registerSessionResourceCleanup, cleanupSessionResources } from "@earendil-works/pi-ai";

registerSessionResourceCleanup("my-extension", async () => {
    await closeOpenAICodexWebSocketSessions();
});

// On shutdown:
await cleanupSessionResources();
```

Introduced in v0.73.0 to give host applications a coordination point for closing long-lived sockets (Codex WebSocket sessions, persistent HTTP keep-alives).

---

## 12. Discrepancies with DeepWiki

Items in the published DeepWiki summary that no longer match v0.74.0:

1. **Xiaomi endpoint.** DeepWiki documents `xiaomi` as routing to `xiaomi-token-plan-ams`. v0.73.0 made this BREAKING — `xiaomi` now hits `api.xiaomimimo.com` directly, and three explicit regional providers exist: `xiaomi-token-plan-cn`, `xiaomi-token-plan-ams`, `xiaomi-token-plan-sgp`.
2. **Codex transport.** DeepWiki lists Codex as SSE-only. v0.74.0 supports `transport: "websocket" | "websocket-cached"` with 5-minute session caching and exported debug helpers (`getOpenAICodexWebSocketDebugStats`, `resetOpenAICodexWebSocketDebugStats`, `closeOpenAICodexWebSocketSessions`).
3. **OAuth providers.** DeepWiki shows `anthropic` only. v0.74.0 ships built-in OAuth for `anthropic`, `openai-codex`, AND `github-copilot` (device flow).
4. **Length-stop overflow.** DeepWiki's overflow detection covers only error messages. v0.74.0 additionally detects Xiaomi MiMo's silent truncation (`stopReason === "length"` + `output === 0` + input ≥ 99% of context window).
5. **Session resource cleanup.** `registerSessionResourceCleanup` / `cleanupSessionResources` are not yet in DeepWiki.
6. **Bedrock bearer token.** `bearerToken` option (and `AWS_BEARER_TOKEN_BEDROCK` env var) bypassing SigV4 is not in DeepWiki.
7. **Azure deployment map.** `AZURE_OPENAI_DEPLOYMENT_NAME_MAP` (comma-separated `model-id=deployment`) is undocumented in DeepWiki.

---

## 13. References

All paths are relative to the pi-mono monorepo at tag `v0.74.0` (commit `1eee081e`).

| Topic | Path |
|---|---|
| Package manifest | `packages/ai/package.json` |
| Root exports | `packages/ai/src/index.ts:1-43` |
| Core types | `packages/ai/src/types.ts` |
| Stream entry points | `packages/ai/src/stream.ts:1-59` |
| API registry | `packages/ai/src/api-registry.ts:1-98` |
| Env var resolution | `packages/ai/src/env-api-keys.ts:1-209` |
| Built-in registration | `packages/ai/src/providers/register-builtins.ts:1-403` |
| Message transformation | `packages/ai/src/providers/transform-messages.ts:1-220` |
| Anthropic provider | `packages/ai/src/providers/anthropic.ts` |
| OpenAI Completions | `packages/ai/src/providers/openai-completions.ts` |
| OpenAI Responses | `packages/ai/src/providers/openai-responses.ts` |
| OpenAI Codex Responses | `packages/ai/src/providers/openai-codex-responses.ts` |
| Azure OpenAI Responses | `packages/ai/src/providers/azure-openai-responses.ts` |
| Google GenAI | `packages/ai/src/providers/google.ts` |
| Google Vertex | `packages/ai/src/providers/google-vertex.ts` |
| Mistral | `packages/ai/src/providers/mistral.ts` |
| Amazon Bedrock | `packages/ai/src/providers/amazon-bedrock.ts` |
| Faux provider | `packages/ai/src/providers/faux.ts:37-126` |
| OAuth registry | `packages/ai/src/utils/oauth/index.ts:1-153` |
| OAuth types | `packages/ai/src/utils/oauth/types.ts:1-72` |
| Anthropic OAuth | `packages/ai/src/utils/oauth/anthropic.ts:31-34` |
| Codex OAuth | `packages/ai/src/utils/oauth/openai-codex.ts:24-28` |
| Event stream | `packages/ai/src/utils/event-stream.ts:1-88` |
| Overflow detection | `packages/ai/src/utils/overflow.ts:1-153` |
| Model catalog | `packages/ai/src/models.generated.ts` |
| Cost / clamping | `packages/ai/src/models.ts:1-92` |
| Quick-start example | `packages/ai/README.md:85-203` |
| Faux example | `packages/ai/README.md:647-715` |
| Cross-provider example | `packages/ai/README.md:910-963` |
| Env var table | `packages/ai/README.md:1031-1056` |
| OAuth section | `packages/ai/README.md:1080-1219` |
| Adding a provider | `packages/ai/README.md:1222-1310` |
| Changelog | `packages/ai/CHANGELOG.md` (v0.73.0 BREAKING xiaomi, v0.73.1 OAuth metadata, v0.74.0) |
