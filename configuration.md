# Configuration

Pi uses a layered configuration system. Project settings override global settings. Environment variables are read at startup.

---

## Configuration Files

| Path | Scope | Purpose |
|------|-------|---------|
| `~/.pi/settings.json` | Global | Default settings for all projects |
| `.pi/settings.json` | Project | Project-specific overrides |
| `~/.pi/agent/auth.json` | Global | API keys and OAuth tokens |
| `~/.pi/agent/models.json` | Global | Custom model definitions and per-provider API keys |

Edit via TUI: `/settings`

Override the agent directory: `PI_CODING_AGENT_DIR=/path/to/dir pi`

---

## settings.json Reference

All keys are optional. Unknown keys are ignored.

### Model

```json
{
  "model": "anthropic/claude-opus-4-6",
  "thinkingLevel": "medium"
}
```

| Key | Type | Description |
|-----|------|-------------|
| `model` | string | Default model (provider/id, fuzzy, or glob) |
| `thinkingLevel` | string | Default thinking: `off`, `minimal`, `low`, `medium`, `high`, `xhigh` |

### Display

```json
{
  "theme": "default",
  "markdown": {
    "codeBlockIndent": 2
  },
  "editorPaddingX": 1,
  "showImages": true,
  "quietStartup": false
}
```

| Key | Type | Description |
|-----|------|-------------|
| `theme` | string | Active theme name |
| `markdown.codeBlockIndent` | number | Code block indentation |
| `editorPaddingX` | number | Left padding in editor |
| `showImages` | boolean | Show inline images (Kitty/iTerm2) |
| `quietStartup` | boolean | Suppress startup messages |

### Session

```json
{
  "treeFilterMode": "all",
  "branchSummary": {
    "skipPrompt": false
  }
}
```

| Key | Type | Description |
|-----|------|-------------|
| `treeFilterMode` | string | Session tree filter: `all`, `bookmarked`, `labeled` |
| `branchSummary.skipPrompt` | boolean | Skip confirmation when creating branch summaries |

### Terminal

```json
{
  "terminal": {
    "clearOnShrink": false
  }
}
```

### Keys

```json
{
  "shellCommandPrefix": "bash -c"
}
```

### Networking

```json
{
  "transport": "auto",
  "retry": {
    "maxDelayMs": 30000
  },
  "cacheRetention": "short"
}
```

| Key | Type | Description |
|-----|------|-------------|
| `transport` | string | `sse`, `websocket`, or `auto` |
| `retry.maxDelayMs` | number | Max retry delay in milliseconds |
| `cacheRetention` | string | `none`, `short`, or `long` |

### Packages

```json
{
  "npmCommand": "npm"
}
```

Override the npm command used for package operations. Useful when using nvm, volta, or other Node version managers.

---

## auth.json

Stores API keys and OAuth tokens. Managed automatically via `/login` and the `models.json` `apiKey` field.

```json
{
  "anthropic": {
    "type": "api_key",
    "apiKey": "sk-ant-..."
  },
  "github-copilot": {
    "type": "oauth",
    "accessToken": "...",
    "refreshToken": "...",
    "expiresAt": 1234567890
  }
}
```

**Security:** File locking prevents concurrent access corruption.

---

## models.json

Customize providers and define custom model entries:

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
      "id": "my-model",
      "name": "My Local Model",
      "provider": "my-local",
      "api": "openai-completions",
      "contextWindow": 128000,
      "maxTokens": 4096,
      "input": ["text"],
      "cost": {
        "input": 0,
        "output": 0,
        "cacheRead": 0,
        "cacheWrite": 0
      }
    }
  }
}
```

API key values prefixed with `!` are resolved as shell commands at runtime.

---

## Context Files

These files shape the agent's behavior and are loaded automatically:

| File | Location | Purpose |
|------|----------|---------|
| `AGENTS.md` | `cwd` and ancestor directories up to git root | Project instructions for the agent |
| `SYSTEM.md` | `~/.pi/agent/` or `.pi/` | Replace the default system prompt entirely |
| `APPEND_SYSTEM.md` | `~/.pi/agent/` or `.pi/` | Append to the default system prompt |

Multiple `AGENTS.md` files are concatenated from outermost to innermost directory.

---

## Skills

Skills are reusable Markdown prompt files. Place them in any of:

```
.agents/skills/     # Project-level (auto-discovered)
.pi/skills/         # Project-level (pi-specific)
~/.pi/skills/       # User-global
~/.agents/skills/   # User-global (shared with other agents)
```

Each skill has a frontmatter header:

```markdown
---
name: code-review
description: Thorough code review checklist and procedure
---

# Code Review Procedure
...
```

Skills appear in the context files system and are available via `/` autocomplete.

---

## Prompt Templates

Prompt templates are Markdown files that can be invoked via slash commands:

```
~/.pi/agent/prompts/    # User-global
.pi/prompts/            # Project-level
```

A template named `standup.md` is invoked with `/standup`.

---

## Themes

Themes are JSON or YAML files defining terminal color schemes:

```
~/.pi/agent/themes/    # User-global
.pi/themes/            # Project-level
```

Switch theme: `/settings` → Theme → select from list

---

## Environment Variables (Full List)

### Provider API Keys

```bash
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_OAUTH_TOKEN=...          # Preferred for OAuth
OPENAI_API_KEY=sk-...
GEMINI_API_KEY=...
GOOGLE_CLOUD_API_KEY=...           # For google-vertex
GROQ_API_KEY=...
CEREBRAS_API_KEY=...
XAI_API_KEY=...
OPENROUTER_API_KEY=...
AI_GATEWAY_API_KEY=...             # Vercel AI Gateway
ZAI_API_KEY=...
MISTRAL_API_KEY=...
MINIMAX_API_KEY=...
HF_TOKEN=...                       # HuggingFace
OPENCODE_API_KEY=...
KIMI_API_KEY=...
COPILOT_GITHUB_TOKEN=...           # GitHub Copilot (also GH_TOKEN or GITHUB_TOKEN)
AWS_ACCESS_KEY_ID=...              # AWS Bedrock
AWS_SECRET_ACCESS_KEY=...
AWS_PROFILE=...                    # AWS profile
```

### Pi Runtime

```bash
PI_CODING_AGENT_DIR=~/.pi/agent    # Override agent directory
PI_OFFLINE=1                        # Disable startup network ops
PI_CACHE_RETENTION=long            # Extended prompt caching
PI_PACKAGE_DIR=/path               # For Nix/Guix environments
PI_SHARE_VIEWER_URL=https://...    # Custom share viewer
PI_TUI_WRITE_LOG=/tmp/pi.log       # Capture raw ANSI stream
PI_DEBUG_REDRAW=1                  # Log redraw triggers
PI_HARDWARE_CURSOR=1               # Enable hardware cursor
PI_CLEAR_ON_SHRINK=1               # Clear on content shrink
```

### AWS Bedrock

```bash
AWS_BEDROCK_SKIP_AUTH=1            # Skip auth (for proxy setups)
AWS_BEDROCK_FORCE_HTTP1=1          # Force HTTP/1.1
```

### Network

```bash
HTTP_PROXY=http://proxy:8080
HTTPS_PROXY=http://proxy:8080
```
