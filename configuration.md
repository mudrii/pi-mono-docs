# Configuration

Pi uses a layered configuration system. Project-local settings (`.pi/`) override global settings (`~/.pi/agent/`). Environment variables are read at startup.

> Authoritative reference: [`packages/coding-agent/docs/settings.md`](https://github.com/earendil-works/pi-mono/blob/v0.74.0/packages/coding-agent/docs/settings.md), [`packages/coding-agent/docs/keybindings.md`](https://github.com/earendil-works/pi-mono/blob/v0.74.0/packages/coding-agent/docs/keybindings.md), [`packages/coding-agent/docs/models.md`](https://github.com/earendil-works/pi-mono/blob/v0.74.0/packages/coding-agent/docs/models.md).

---

## Configuration Files

| Path | Scope | Purpose |
|------|-------|---------|
| `~/.pi/agent/settings.json` | Global | Default settings for all projects |
| `.pi/settings.json` | Project | Project-specific overrides (merged shallowly) |
| `~/.pi/agent/auth.json` | Global | API keys and OAuth tokens (0600 perms) |
| `~/.pi/agent/models.json` | Global | Custom providers and model definitions (JSONC since v0.73.1) |
| `~/.pi/agent/keybindings.json` | Global | Keybinding overrides |
| `.pi/keybindings.json` | Project | Project keybinding overrides |
| `~/.pi/agent/SYSTEM.md` / `APPEND_SYSTEM.md` | Global | Replace / extend system prompt |
| `.pi/SYSTEM.md` / `APPEND_SYSTEM.md` | Project | Replace / extend system prompt for this project |

Edit settings via TUI: `/settings`. Reload after edits: `/reload`.

Override the agent directory: `PI_CODING_AGENT_DIR=/path/to/dir pi`.

The directory name is read from the `piConfig.configDir` field of `package.json` (default: `.pi`).

---

## settings.json Reference

All keys are optional. Unknown keys are ignored. The complete authoritative list lives in `packages/coding-agent/docs/settings.md`; common keys are summarized below.

### Model and Thinking

```json
{
  "model": "anthropic/claude-opus-4-7",
  "thinkingLevel": "medium",
  "thinkingBudgets": {
    "minimal": 1024,
    "low": 4096,
    "medium": 16384,
    "high": 32768,
    "xhigh": 65536
  }
}
```

| Key | Type | Description |
|-----|------|-------------|
| `model` | string | Default model (provider/id, fuzzy, glob, or `name:thinking`) |
| `thinkingLevel` | string | `off`, `minimal`, `low`, `medium`, `high`, `xhigh` |
| `thinkingBudgets` | object | Per-level token budget overrides |

### Tools

```json
{
  "tools": ["read", "bash", "edit", "write"],
  "enabledTools": ["read", "grep", "find", "ls"],
  "shellPath": "/bin/bash",
  "shellCommandPrefix": ["bash", "-lc"]
}
```

| Key | Type | Description |
|-----|------|-------------|
| `tools` | string[] | Allowlist of tool names (built-in, extension, custom) |
| `shellPath` | string | Shell binary used by `bash` tool |
| `shellCommandPrefix` | string[] | argv prefix wrapping every `bash` command |

### Display

```json
{
  "theme": "default",
  "markdown": { "codeBlockIndent": 2 },
  "editorPaddingX": 1,
  "showImages": true,
  "quietStartup": false,
  "autocompleteMaxVisible": 8,
  "showHardwareCursor": false
}
```

### Session

```json
{
  "sessionDir": "~/.pi/agent/sessions",
  "treeFilterMode": "all",
  "branchSummary": {
    "skipPrompt": false,
    "reserveTokens": 16384
  },
  "doubleEscapeAction": "interrupt"
}
```

| Key | Type | Description |
|-----|------|-------------|
| `sessionDir` | string | Session directory (`~` is expanded) |
| `treeFilterMode` | string | `all`, `bookmarked`, or `labeled` |
| `branchSummary.skipPrompt` | boolean | Skip confirm when summarizing an abandoned branch |
| `doubleEscapeAction` | string | What double-Escape does in the editor |

> **Note (v0.65.0):** the `session_directory` setting was removed. Use `sessionDir`, `--session-dir`, `PI_CODING_AGENT_SESSION_DIR`, or `PI_CODING_AGENT_DIR`.

### Compaction (Authoritative Defaults)

```json
{
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  }
}
```

| Key | Default | Description |
|-----|---------|-------------|
| `compaction.enabled` | `true` | Auto-compact when context fills |
| `compaction.reserveTokens` | `16384` | Tokens reserved for LLM response |
| `compaction.keepRecentTokens` | `20000` | Recent tokens kept verbatim (not summarized) |

### Networking and Retry

```json
{
  "transport": "auto",
  "cacheRetention": "short",
  "retry": {
    "maxDelayMs": 30000,
    "maxAttempts": 8,
    "provider": {
      "anthropic": { "maxAttempts": 12 }
    }
  }
}
```

| Key | Type | Description |
|-----|------|-------------|
| `transport` | string | `sse`, `websocket`, `websocket-cached`, or `auto` |
| `cacheRetention` | string | `none`, `short`, or `long` |
| `retry.*` | object | Global retry policy |
| `retry.provider.<id>.*` | object | Per-provider retry overrides |

### Terminal and Images

```json
{
  "terminal": {
    "clearOnShrink": false,
    "imageWidthCells": 60
  },
  "images": {
    "maxWidth": 1600
  }
}
```

### Packages and Resources

```json
{
  "npmCommand": ["npm"],
  "packages": ["@scope/some-pi-extension", { "name": "foo", "enabled": false }],
  "extensions": ["~/code/my-ext/index.ts"],
  "skills": ["~/notes/skills"],
  "prompts": ["~/notes/prompts"],
  "themes": ["~/.pi/agent/themes"],
  "enableSkillCommands": true,
  "enabledModels": ["anthropic/claude-*", "openai/gpt-5*"],
  "enableInstallTelemetry": false,
  "warnings": { "anthropicExtraUsage": true }
}
```

`npmCommand` is an **argv array** (since v0.66+); a bare string is also accepted for backwards compatibility. Use this to bind pi to a particular Node version manager:

```json
{ "npmCommand": ["volta", "run", "npm"] }
```

---

## auth.json

Stores API keys and OAuth tokens. Managed automatically via `/login` and the `models.json` `apiKey` field. Created with `0600` permissions.

```json
{
  "anthropic": { "type": "api_key", "key": "sk-ant-..." },
  "openai":    { "type": "oauth", "access_token": "...", "refresh_token": "...", "expires_at": 1234567890 },
  "github-copilot": { "type": "oauth", "access_token": "...", "refresh_token": "...", "expires_at": 1234567890 }
}
```

Auth file values take priority over environment variables. A `key` value supports three resolution forms:

| Form | Example |
|------|---------|
| Shell command | `"!op read 'op://vault/anthropic/credential'"` (cached for process lifetime) |
| Env var name | `"MY_ANTHROPIC_KEY"` |
| Literal | `"sk-ant-..."` |

File locking via `proper-lockfile` prevents concurrent corruption.

---

## models.json

Customize providers and define custom model entries. Since v0.73.1, `models.json` accepts JSONC-style comments and trailing commas.

```json
{
  "providers": {
    "anthropic": {
      "apiKey": "sk-ant-...",
      "baseUrl": "https://api.anthropic.com"
    },
    "openai": {
      "apiKey": "!op read op://vault/openai/api-key"
    },
    "my-local": {
      "api": "openai-completions",
      "baseUrl": "http://localhost:11434/v1",
      "apiKey": "dummy"
    }
  },
  "models": {
    "my-model": {
      "id": "llama3",
      "name": "Llama 3 (Local)",
      "provider": "my-local",
      "api": "openai-completions",
      "contextWindow": 128000,
      "maxTokens": 4096,
      "input": ["text"],
      "reasoning": false,
      "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 }
    }
  }
}
```

`apiKey` and entries in `headers` support the same three resolution forms as `auth.json`. The 4 user-selectable `api:` values for a custom provider are `openai-completions`, `openai-responses`, `anthropic-messages`, and `google-generative-ai`.

See `packages/coding-agent/docs/models.md` for every field including `modelOverrides`, `compat.*`, `thinkingLevelMap`, and `authHeader`.

---

## keybindings.json

Keybinding actions live in a single namespace (`tui.editor.*`, `app.*`, etc.). Override per action with one key or a list:

```json
{
  "app.interrupt": "ctrl+c",
  "app.session.tree": ["ctrl+t", "F2"],
  "tui.editor.cursorUp": "up",
  "app.thinking.cycle": "alt+t",
  "app.model.cycleForward": "alt+m",
  "app.message.followUp": "alt+enter"
}
```

Reload with `/reload`. See `packages/coding-agent/docs/keybindings.md` for the full action catalog.

---

## Context Files

These files shape the agent's behavior and are loaded automatically:

| File | Location | Purpose |
|------|----------|---------|
| `AGENTS.md` | `cwd` and ancestor directories up to git root | Project instructions for the agent |
| `CLAUDE.md` | Same discovery rule | Alias for `AGENTS.md` (concatenated alongside) |
| `SYSTEM.md` | `~/.pi/agent/` or `.pi/` | **Replace** the default system prompt entirely |
| `APPEND_SYSTEM.md` | `~/.pi/agent/` or `.pi/` | Append to the default system prompt |

Multiple `AGENTS.md` / `CLAUDE.md` files are concatenated from outermost to innermost directory. See [context.md](context.md).

---

## Resource Directories

Pi auto-discovers user-defined resources at both project and user-global level:

```
.pi/extensions/    ~/.pi/agent/extensions/    Project / user TypeScript extensions
.pi/skills/        ~/.pi/agent/skills/        Markdown skills
.agents/skills/                               Cross-tool skills (also discovered)
.pi/prompts/       ~/.pi/agent/prompts/       Slash-command prompt templates
.pi/themes/        ~/.pi/agent/themes/        Theme files (JSON or YAML)
```

Project resources take precedence over user-global. See [extensions.md](extensions.md) and [context.md](context.md).

---

## Environment Variables

### Provider API Keys (excerpt — full list in [providers.md](providers.md))

```bash
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_OAUTH_TOKEN=...          # Preferred over API key for Anthropic
OPENAI_API_KEY=sk-...
GEMINI_API_KEY=...
GOOGLE_CLOUD_API_KEY=...           # google-vertex
GROQ_API_KEY=...                   CEREBRAS_API_KEY=...
XAI_API_KEY=...                    OPENROUTER_API_KEY=...
AI_GATEWAY_API_KEY=...             # Vercel AI Gateway
ZAI_API_KEY=...                    MISTRAL_API_KEY=...
MINIMAX_API_KEY=...                MINIMAX_CN_API_KEY=...
MOONSHOT_API_KEY=...               # moonshotai and moonshotai-cn
HF_TOKEN=...                       FIREWORKS_API_KEY=...
OPENCODE_API_KEY=...               # opencode and opencode-go
KIMI_API_KEY=...                   DEEPSEEK_API_KEY=...
COPILOT_GITHUB_TOKEN=...           # also GH_TOKEN or GITHUB_TOKEN
CLOUDFLARE_API_KEY=...             # workers-ai and ai-gateway
CLOUDFLARE_ACCOUNT_ID=...          CLOUDFLARE_GATEWAY_ID=...
XIAOMI_API_KEY=...                 # mimo
XIAOMI_TOKEN_PLAN_CN_API_KEY=...
XIAOMI_TOKEN_PLAN_AMS_API_KEY=...
XIAOMI_TOKEN_PLAN_SGP_API_KEY=...
AWS_ACCESS_KEY_ID=... AWS_SECRET_ACCESS_KEY=...   # amazon-bedrock
AWS_PROFILE=... AWS_BEARER_TOKEN_BEDROCK=...      # alternatives
AZURE_OPENAI_API_KEY=... AZURE_OPENAI_BASE_URL=...
```

### Pi Runtime

```bash
PI_CODING_AGENT_DIR=~/.pi/agent            # Override the whole agent dir
PI_CODING_AGENT_SESSION_DIR=/path/to/sessions   # v0.71.0
PI_OFFLINE=1                                # Disable startup network ops
PI_CACHE_RETENTION=long                     # Anthropic extended prompt cache
PI_PACKAGE_DIR=/path                        # Writable package dir for Nix/Guix
PI_SKIP_VERSION_CHECK=1                     # Suppress update check
PI_TELEMETRY=0                              # Disable optional telemetry
PI_SHARE_VIEWER_URL=https://...             # Custom share viewer
PI_TUI_WRITE_LOG=/tmp/pi.log                # Capture raw ANSI stream
PI_DEBUG_REDRAW=1                           # Log redraw triggers
PI_HARDWARE_CURSOR=1                        # Hardware cursor
PI_CLEAR_ON_SHRINK=1                        # Clear screen on shrink
PI_OAUTH_CALLBACK_HOST=0.0.0.0              # Bind OAuth callback (v0.68.0)
VISUAL=$EDITOR                              # Used for `$EDITOR` slash command
```

### AWS Bedrock Overrides

```bash
AWS_BEDROCK_SKIP_AUTH=1            # Skip auth (proxy in front)
AWS_BEDROCK_FORCE_HTTP1=1          # Force HTTP/1.1
AWS_BEDROCK_FORCE_CACHE=1          # Force prompt caching for app inference profiles
AWS_ENDPOINT_URL_BEDROCK_RUNTIME=https://my-proxy/bedrock
```

### Network

```bash
HTTP_PROXY=http://proxy:8080
HTTPS_PROXY=http://proxy:8080
```

---

## Source-of-Truth Pointers

- `packages/coding-agent/src/core/settings-manager.ts` — settings shape and defaults
- `packages/coding-agent/docs/settings.md` — full settings reference (first-party)
- `packages/coding-agent/docs/models.md` — `models.json` schema
- `packages/coding-agent/docs/keybindings.md` — keybinding registry
- `packages/ai/src/env-api-keys.ts` — `envMap` (provider → env var)
