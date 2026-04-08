# LLM Providers

Pi includes 23+ LLM providers with 100+ models. Only models supporting tool calling are included in the catalog. (Updated for v0.65.2.)

---

## Provider Overview

| Provider ID | Auth Method | Notes |
|-------------|-------------|-------|
| `anthropic` | `ANTHROPIC_API_KEY` or OAuth | Claude family; Opus 4.6 default (1M context) |
| `openai` | `OPENAI_API_KEY` | GPT-5.4 default |
| `openai-codex` | OAuth or `OPENAI_API_KEY` | Responses/WebSocket; GPT-5.3-codex, GPT-5.4, GPT-5.4-mini (v0.61.0) |
| `azure-openai-responses` | Azure credentials | Azure OpenAI Responses API |
| `google` | `GEMINI_API_KEY` | Gemini 3.x |
| `google-vertex` | ADC or `GOOGLE_CLOUD_API_KEY` | Vertex AI; includes `gemini-3.1-pro-preview-customtools` (v0.63.1) |
| `google-gemini-cli` | OAuth | Gemini CLI; 1M context |
| `google-antigravity` | Internal OAuth | Claude Sonnet/Opus 4.6 Thinking |
| `github-copilot` | OAuth device code | Claude 4.x via Anthropic Messages API |
| `amazon-bedrock` | `AWS_ACCESS_KEY_ID` / `AWS_PROFILE` | Bedrock Converse Stream |
| `openrouter` | `OPENROUTER_API_KEY` | 100+ models; `openrouter:auto` alias |
| `vercel-ai-gateway` | `AI_GATEWAY_API_KEY` | Provider failover and load balancing |
| `groq` | `GROQ_API_KEY` | Qwen3 with reasoning effort mapping |
| `cerebras` | `CEREBRAS_API_KEY` | — |
| `xai` | `XAI_API_KEY` | Grok models |
| `mistral` | `MISTRAL_API_KEY` | Native Mistral SDK |
| `minimax` | `MINIMAX_API_KEY` | MiniMax — supported model IDs: `MiniMax-M2.7`, `MiniMax-M2.7-highspeed` (v0.63.0: legacy `minimax`/`minimax-cn` direct IDs removed) |
| `huggingface` | `HF_TOKEN` | OpenAI-compatible Inference Router |
| `opencode` | `OPENCODE_API_KEY` | OpenCode |
| `opencode-go` | `OPENCODE_API_KEY` | OpenCode Go |
| `kimi-coding` | `KIMI_API_KEY` | Moonshot AI (Anthropic-compatible) |
| `zai` | `ZAI_API_KEY` | z.ai GLM-5 |

---

## Setting Up Providers

### Environment Variables (Simplest)

```bash
export ANTHROPIC_API_KEY=sk-ant-...
export OPENAI_API_KEY=sk-...
export GEMINI_API_KEY=...
export OPENROUTER_API_KEY=sk-or-...
pi
```

### Via models.json

```json
// ~/.pi/agent/models.json
{
  "providers": {
    "anthropic": { "apiKey": "sk-ant-..." },
    "openai": { "apiKey": "!op read op://vault/openai/api-key" },
    "openrouter": { "apiKey": "sk-or-..." }
  }
}
```

Values starting with `!` are evaluated as shell commands at runtime.

### OAuth Login (Interactive)

```bash
pi
/login
```

Select from: Anthropic (Claude Pro/Max), GitHub Copilot, Google Gemini CLI, Google Antigravity, OpenAI Codex.

---

## Model Selection

### Default Models

| Provider | Default Model |
|----------|--------------|
| `anthropic` | `claude-opus-4-6` (1M context) |
| `openai` | `gpt-5.4` |
| `openai-codex` | `gpt-5.4` |
| `google` | `gemini-3-pro` |

### Selection Formats

```bash
# Exact model ID
pi --model claude-opus-4-6

# Provider-qualified
pi --model anthropic/claude-opus-4-6

# Fuzzy match
pi --model opus
pi --model sonnet

# Glob pattern
pi --model '*sonnet*'

# With thinking level
pi --model sonnet:high
pi --model anthropic/opus:medium

# Mid-session
/model anthropic/claude-sonnet-4-6
```

---

## Thinking / Reasoning

Thinking levels abstract over provider-native reasoning:

| Level | Anthropic | OpenAI | Notes |
|-------|-----------|--------|-------|
| `off` | No thinking | No reasoning | Default for most models |
| `minimal` | Low budget | Low effort | — |
| `low` | Low budget | Low effort | — |
| `medium` | Medium budget | Medium effort | — |
| `high` | High budget | High effort | — |
| `xhigh` | Max budget | Max effort | GPT-5.2+, Claude Opus 4.6 |

Set default in settings:
```json
{ "thinkingLevel": "medium" }
```

Set per-session: `pi --thinking high` or `/thinking xhigh`

Set per-model: `pi --model sonnet:medium`

---

## OpenRouter

Access 100+ models through a single API key:

```bash
export OPENROUTER_API_KEY=sk-or-...
pi --model openrouter/anthropic/claude-opus-4-6
pi --model openrouter:auto     # Let OpenRouter auto-select
```

Configure routing in `models.json`:

```json
{
  "providers": {
    "openrouter": {
      "apiKey": "sk-or-...",
      "routing": {
        "only": ["anthropic", "openai"],
        "order": ["anthropic", "openai"]
      }
    }
  }
}
```

---

## Vercel AI Gateway

Multi-provider failover and load balancing:

```bash
export AI_GATEWAY_API_KEY=...
pi --model vercel-ai-gateway/...
```

---

## AWS Bedrock

```bash
# Via environment
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...

# Or via AWS profile
export AWS_PROFILE=my-profile

# For proxy setups
AWS_BEDROCK_SKIP_AUTH=1 pi
AWS_BEDROCK_FORCE_HTTP1=1 pi   # Force HTTP/1.1
```

Supports prompt caching for Claude models on Bedrock (v0.58.1+).

---

## Google Vertex AI

```bash
# Application Default Credentials (gcloud auth application-default login)
pi --model google-vertex/...

# Or via API key
export GOOGLE_CLOUD_API_KEY=...
pi --model google-vertex/...
```

---

## Local Models (LM Studio / Ollama)

Add a custom provider in `models.json`:

```json
{
  "providers": {
    "my-local": {
      "api": "openai-completions",
      "baseUrl": "http://localhost:11434/v1",
      "apiKey": "ollama"
    }
  },
  "models": {
    "llama3-local": {
      "id": "llama3",
      "name": "Llama 3 (Local)",
      "provider": "my-local",
      "api": "openai-completions",
      "contextWindow": 8192,
      "maxTokens": 4096,
      "input": ["text"],
      "reasoning": false,
      "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 }
    }
  }
}
```

Then: `pi --model llama3-local`

---

## Custom Providers (Extensions)

Extensions can register entirely custom providers at runtime:

```typescript
pi.registerProvider({
  id: "my-provider",
  name: "My Provider",
  streamSimple: async (model, context, options) => {
    // Custom implementation
    return customStream(model, context, options);
  }
});
```

Remove dynamically:
```typescript
pi.unregisterProvider("my-provider");
```

See examples in `packages/coding-agent/examples/extensions/`.

---

## Transport Options

```json
{ "transport": "sse" }       // Force Server-Sent Events
{ "transport": "websocket" } // Force WebSocket
{ "transport": "auto" }      // Let provider decide (default)
```

OpenAI Codex uses WebSocket by default.

---

## Prompt Caching

Anthropic and some other providers support prompt caching. Enable extended retention:

```bash
PI_CACHE_RETENTION=long pi
```

Or in settings:
```json
{ "cacheRetention": "long" }
```

Cache costs (Anthropic): cacheWrite at ~25% of input rate, cacheRead at ~10% of input rate. Significant savings on long sessions.
