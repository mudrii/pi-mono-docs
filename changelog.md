# Changelog

All current pi-mono releases follow lockstep versioning for the five active packages (`pi-ai`, `pi-tui`, `pi-agent-core`, `pi-coding-agent`, `pi-web-ui`). Starting with v0.74.0, active packages are published under `@earendil-works/*`. The previous `@mariozechner/*` packages remain published for older installs and stop at v0.73.1. `@mariozechner/pi-mom` and `@mariozechner/pi-pods` were removed from the monorepo at v0.71.0.

> **Released vs Unreleased** — v0.74.0 (2026-05-07) is the latest released cut. Anything on `main` after that commit is unreleased and is documented separately in [Era 9](./09-version-history.md#main-after-v0740-unreleased) of the detailed version history.

This document is the concise top-level changelog. For per-release breakdowns with PR/issue links and granular package-level notes, see [`09-version-history.md`](./09-version-history.md).

---

## v0.73.0-v0.74.0 (2026-05-04 to 2026-05-07)

### v0.74.0 - 2026-05-07
- **Release-wide**: Updated repository links and package references for the move to `earendil-works/pi-mono` and `@earendil-works/*` package scopes.

### v0.73.1 - 2026-05-07
- `coding-agent`: Added self-update support for the package scope migration.
- `coding-agent`: Added interactive OAuth login choices.
- `coding-agent`: Changed extension loading to upstream `jiti` 2.7.
- `coding-agent`: Changed `models.json` parsing to allow comments and trailing commas.
- `coding-agent` / `ai`: Fixed OpenAI Codex OAuth stderr writes, Codex non-empty system prompts, interleaved content/tool-call deltas, and Kimi K2 P6 alias resolution.
- `tui`: Fixed OSC 8 hyperlinks and Kitty inline image rendering edge cases.

### v0.73.0 - 2026-05-04
- **Breaking** `ai` / `coding-agent`: Switched `xiaomi` from Token Plan AMS to API billing; Token Plan users must use `xiaomi-token-plan-cn`, `xiaomi-token-plan-ams`, or `xiaomi-token-plan-sgp`.
- `ai`: Added session-scoped resource cleanup helpers.
- `ai`: Fixed OpenCode Go model metadata, Bedrock Claude Opus 4.7 `xhigh`, and Codex WebSocket fallback diagnostics.
- `coding-agent`: Added incremental bash output streaming and compact `read` rendering.
- `tui`: Fixed fuzzy ranking to prioritize exact matches.

## v0.70.0–v0.72.1 (2026-04-23 to 2026-05-02)

### v0.72.1 — 2026-05-02
- `agent`: Changed default transport to `auto` so providers use their best available transport
- `ai`: Fixed OpenAI Codex transport option being ignored (always used SSE)
- `ai`: Updated generated models catalog
- `coding-agent`: Fixed compact read rendering — Pi docs, AGENTS/CLAUDE context files, and SKILL.md collapsed by default in interactive output
- `coding-agent`: Fixed Codex WebSocket sessions kept alive after `--print` / JSON mode ends

### v0.72.0 — 2026-05-01 *(Breaking)*
- **Breaking** `ai`: Replaced `compat.reasoningEffortMap` with top-level `Model.thinkingLevelMap`; removed `supportsXhigh()`, use `getSupportedThinkingLevels()`/`clampThinkingLevel()` instead
- `ai`: Added Xiaomi MiMo Token Plan provider (`XIAOMI_API_KEY`, `mimo-v2.5-pro` default)
- `ai`: Added `getSupportedThinkingLevels()` and `clampThinkingLevel()` functions
- `agent`: Added `shouldStopAfterTurn` callback to agent loop config for graceful post-turn exit
- `coding-agent`: Added Xiaomi MiMo Token Plan to `/login` and default model resolution
- `coding-agent`: Fixed custom provider `pi.registerProvider()` to honor per-model `baseUrl` overrides
- `coding-agent`: Fixed self-update detection

### v0.71.1 — 2026-05-01
- `ai`: Added `websocket-cached` transport for OpenAI Codex Responses (ChatGPT subscription auth)
- `coding-agent`: Added `websocket-cached` transport option for OpenAI Codex

### v0.71.0 — 2026-04-30 *(Breaking)*
- **Breaking** `ai` / `coding-agent`: Removed Google Gemini CLI and Google Antigravity providers completely
- **Removed** `mom` and `pods` packages from monorepo
- `ai`: Added Cloudflare AI Gateway provider (`CLOUDFLARE_API_KEY`/`CLOUDFLARE_ACCOUNT_ID`/`CLOUDFLARE_GATEWAY_ID`)
- `ai`: Added Moonshot AI provider (`MOONSHOT_API_KEY`)
- `ai`: Added Mistral Medium 3.5 model
- `ai`: Added `AssistantMessage.responseModel` (exposes routed model on openai-completions path)
- `ai`: Fixed Google Vertex unsigned tool call replay
- `ai`: Updated @anthropic-ai/sdk to ^0.91.1 (security: GHSA-p7fg-763f-g4gf)
- `coding-agent`: Added `PI_CODING_AGENT_SESSION_DIR` env var (equivalent to `--session-dir`)
- `coding-agent`: Added `message_end` extension result support (replace finalized messages)
- `coding-agent`: Added `ctx.ui.getEditorComponent()` for extensions
- `coding-agent`: Added `thinking_level_select` extension event
- `coding-agent`: Added top-level `name` field to `pi.registerProvider()` for `/login` display
- `coding-agent`: Fixed WSL clipboard image paste
- `coding-agent`: Fixed PowerShell Windows stdio (`detached: false` on Windows)
- `coding-agent`: Fixed `AGENTS.MD` uppercase context file discovery
- `coding-agent`: Fixed Bun node_modules discovery
- `coding-agent`: Fixed `/handoff` to use compacted context

### v0.70.6 — 2026-04-28
- `ai`: Added Cloudflare Workers AI provider (`CLOUDFLARE_API_KEY`/`CLOUDFLARE_ACCOUNT_ID`)
- `coding-agent`: Pi update checks use `pi.dev` and `pi/<version>` user agent
- `coding-agent`: Fixed exported HTML XSS (escapes embedded image data and session metadata)
- `coding-agent`: Fixed Bun startup (global node_modules relative to Bun install layout)
- `coding-agent`: Fixed `pi update --self` for Windows shim installs

### v0.70.5 — 2026-04-27
- `coding-agent`: Fixed HTML export ANSI trailing padding

### v0.70.4 — 2026-04-27
- `coding-agent`: Fixed packaged `pi` startup failure (session selector source import)

### v0.70.3 — 2026-04-27
- `ai`: Added Azure Cognitive Services endpoint for Azure OpenAI Responses
- `ai`: Fixed empty tools array sent when no tools active (fixes DashScope/Aliyun Qwen 400)
- `ai`: Fixed Bedrock inference profile ARN capability checks

### v0.70.2 — 2026-04-24
- `ai`: Fixed OpenAI/Azure/Anthropic request options forwarding (omit undefined timeout/maxRetries)

### v0.70.1 — 2026-04-24
- `ai`: Added DeepSeek as built-in provider (V4 Flash, V4 Pro, `DEEPSEEK_API_KEY`)
- `ai`: Fixed DeepSeek V4 session replay 400 errors (reasoning compat)
- `ai`: Fixed GPT-5.5 context window metadata (272k)
- `ai`: Exposed `timeoutMs` and `maxRetries` in stream options

### v0.70.0 — 2026-04-23
- `ai`: Added GPT-5.5 to OpenAI Codex model generation
- `ai`: Added `findEnvKeys()` for identifying configured provider env vars
- `ai`: Fixed Google Vertex custom `model.baseUrl` forwarding
- `ai`: Fixed OpenAI-compatible usage parsing (no double-counting reasoning tokens)
- `ai`: Fixed tool-call coalescing for mutating gateway IDs (fixes Kimi K2.6/OpenCode)

---

## v0.67.0–v0.69.0 (2026-04-13 to 2026-04-22)

### Breaking Changes
- **v0.68.0**: Tool selection API changed from `Tool[]` to `string[]`; prebuilt tool exports removed; use factory functions
- **v0.68.0**: `DefaultResourceLoader`/`loadProjectContextFiles()`/`loadSkills()` require explicit `cwd`
- **v0.68.0**: `--no-tools` now disables ALL tools (was built-ins only)
- **v0.69.0**: TypeBox 1.x migration (import from `typebox`, not `@sinclair/typebox`)
- **v0.69.0**: Session-replacement callbacks invalidate pre-switch objects (use `withSession`)

### New Features
- Fireworks provider (`FIREWORKS_API_KEY`)
- Stacked autocomplete providers (`ctx.ui.addAutocompleteProvider`)
- Terminating tool results (`terminate: true`)
- OSC 9;4 terminal progress indicators
- Configurable working indicator (`ctx.ui.setWorkingIndicator`)
- `/clone` command (duplicate active session)
- `systemPromptOptions` in `before_agent_start`
- `session_shutdown` reasons and `targetSessionFile`
- Bedrock bearer-token auth (`AWS_BEARER_TOKEN_BEDROCK`)
- `claude-opus-4-7` model added
- `onResponse` in `StreamOptions`
- `thinkingDisplay` option (summarized/omitted)
- Per-tool `executionMode` override
- Configurable keybindings for model selector and tree filter

---

## v0.66.1 (2026-04-08)
- **coding-agent**: Changed Earendil announcement from automatic startup notice to hidden `/dementedelves` slash command

## v0.66.0 (2026-04-08)
- **coding-agent**: Earendil startup announcement with bundled inline image rendering; interactive Anthropic subscription auth warning
- **ai**: Fixed bare `readline` import to use `node:readline` prefix for Deno compatibility (#2885)
- **coding-agent**: Fixed auto-retry for stream failures (#2892); Deno compatibility fix

---

## v0.65.2 — 2026-04-06

**Features:**
- `tui`: TUI render scheduling throttle to reduce unnecessary redraws
- `coding-agent`: Earendil startup announcement with bundled inline image rendering and linked blog post for April 8–9, 2026
- `coding-agent`: Interactive Anthropic subscription auth warning when subscription auth is active, clarifying that third-party usage draws from extra usage and is billed per token
- `ai`: Model catalog update

---

## v0.65.1 — 2026-04-05

**Bug fixes:**
- `coding-agent`: Fixed bash output line-truncation to always persist full output to a temp file (prevents data loss when output exceeds 2000 lines but stays under the byte threshold)
- `coding-agent`: RPC client now forwards subprocess stderr to parent process in real-time
- `coding-agent`: Theme file watcher now handles async `fs.watch` error events instead of crashing the process
- `coding-agent`: Fixed stored session cwd handling; resuming a session whose original working directory no longer exists now prompts interactive users to continue in the current cwd, while non-interactive modes fail with a clear error
- `coding-agent`: Fixed resource collision precedence so project and user skills, prompt templates, and themes override package resources consistently
- `coding-agent`: Fixed CLI extension paths (e.g. `git:gist.github.com/...`) being incorrectly resolved against cwd instead of being passed through to the package manager
- `coding-agent`: Fixed piped stdin runs with `--mode json` to preserve JSONL output instead of falling back to plain text

---

## v0.65.0 — 2026-04-03

**[BREAKING]**
- Removed extension post-transition events `session_switch` and `session_fork`. Use `session_start` with `event.reason` (`"startup" | "reload" | "new" | "resume" | "fork"`).
- Removed session-replacement methods from `AgentSession`. Use `AgentSessionRuntime` for `newSession()`, `switchSession()`, `fork()`, and `importFromJsonl()`.
- Removed `session_directory` from extension and settings APIs.
- Unknown single-dash CLI flags (e.g. `-s`) now produce an error instead of being silently ignored.

**Features:**
- `coding-agent`: `createAgentSessionRuntime()` and `AgentSessionRuntime` for closure-based runtime that recreates cwd-bound services on every session switch
- `coding-agent`: `defineTool()` helper for standalone custom tool definitions with full TypeScript parameter type inference
- `coding-agent`: Unified diagnostics (`info`/`warning`/`error`) for arg parsing, service creation, session option resolution, and resource loading
- `tui`: Toggle timestamps on `/tree` entries with `Shift+T` (smart date formatting, preserved through branching)

---

## v0.64.0 — 2026-03-29

**[BREAKING]**
- `ModelRegistry` no longer has a public constructor. Use `ModelRegistry.create(authStorage, modelsJsonPath?)` or `ModelRegistry.inMemory(authStorage)`.

**Features:**
- `coding-agent`: `prepareArguments` hook on tool definitions lets extensions normalize or migrate raw model arguments before schema validation; the built-in `edit` tool uses this to transparently support sessions with the old single-edit schema
- `coding-agent`: Extensions can customize the collapsed thinking block label via `ctx.ui.setHiddenThinkingLabel()`
- `coding-agent`: Faux provider support

---

## v0.63.2 — 2026-03-29

**Features:**
- `coding-agent`: `ctx.signal` added to `ExtensionContext` — forwards cancellation into nested model calls and `fetch()`
- `coding-agent`: Built-in `edit` tool unified to `edits[]` as the only replacement shape (legacy top-level `oldText`/`newText` removed)

**Bug fixes:**
- `tui`: Fixed TUI keyboard handling and large multi-edit diff rendering

---

## v0.63.1 — 2026-03-27

**Features:**
- `ai`: Added `gemini-3.1-pro-preview-customtools` model for the `google-vertex` provider

**Bug fixes:**
- `coding-agent`: Overflow detection improvements for repeated compactions

---

## v0.63.0 — 2026-03-27

**[BREAKING]**
- `ModelRegistry.getApiKey(model)` replaced by `getApiKeyAndHeaders(model)`. Extensions and SDK integrations must now fetch request auth per call and forward both `apiKey` and `headers`.
- Removed deprecated direct `minimax` and `minimax-cn` model IDs. Supported direct MiniMax IDs are now `MiniMax-M2.7` and `MiniMax-M2.7-highspeed`.

**Features:**
- `coding-agent`: `PI_TUI_WRITE_LOG` environment variable to capture raw ANSI stream for debugging

---

## v0.62.0 — 2026-03-23

**Features:**
- `ai`: `BedrockOptions.requestMetadata` support for attaching custom metadata to Bedrock requests

**Bug fixes:**
- `ai`: Fixed thinking disable to work correctly across model switches
- `tui`: Fixed TUI truncation for wide-character content

---

## v0.61.1 — 2026-03-20

**Features:**
- `ai`: MiniMax metadata support

**Bug fixes:**
- `tui`: TUI keybinding fix
- `coding-agent`: Termux compatibility fix

---

## v0.61.0 — 2026-03-20

**Features:**
- `ai`: Added `gpt-5.4-mini` model for the `openai-codex` provider
- `coding-agent`: `validateToolArguments` fallback for graceful degradation on invalid tool arguments
- `ai`: Amazon Bedrock inference profiles support
- `ai`: OpenRouter reasoning fix

---

## v0.58.4 — 2026-03-16

**Bug fixes:**
- `agent` / `coding-agent`: Fixed steering messages to wait until the current assistant message's tool-call batch fully finishes instead of skipping pending tool calls.

**Features:**
- `web-ui`: Added `onModelSelect` callback on `AgentInterface` and `ChatPanel.setAgent` config
- `web-ui`: Added `allowedProviders` filter on `ModelSelector.open()` to restrict visible models
- `web-ui`: Added `onClose` callback on `SettingsDialog.open()`
- `web-ui`: Added `state_change` event emitted by `Agent` on `setModel()` and `setThinkingLevel()`
- `web-ui`: Subsequence-based fuzzy search in model selector (replaces substring matching)
- `web-ui`: Added `openai-codex` and `github-copilot` to `shouldUseProxyForProvider`

---

## v0.58.3 — 2026-03-15

**Bug fixes:**
- `web-ui`: Build with tsc instead of tsgo
- `coding-agent`: Normalize fuzzy edit matching and run touched tests
- `coding-agent`: Handle slash-delimited `/model` refs correctly
- `ai`: Fix Anthropic OAuth manual callback and refresh flow

---

## v0.58.2 — 2026-03-15

**Bug fixes:**
- `tui`: Clear stale scrollback on session switch
- `tui`: Remove trailing markdown block spacing
- `ai`: Align Codex WebSocket headers and terminate SSE properly
- `coding-agent`: Handle WSL clipboard image fallback
- `coding-agent`: Start UI before `session_start` event
- `coding-agent`: Stabilize Windows shell/path handling
- `ai`: Limit Bedrock prompt caching to Claude models
- `ai`: Add Qwen chat-template compatibility mode
- `ai`: Harden Bedrock unsigned thinking replay
- `coding-agent`: Normalize prompt cwd for bash-safe Windows paths
- `coding-agent`: Support configurable npm wrapper command
- `tui`: Preserve literal paste content
- `coding-agent`: Stop skill recursion at root
- `tui`: Preserve `./` prefix in tab completion
- `coding-agent`: Fix startup crash when downloading fd/ripgrep on first run

**Features:**
- `tui`: Configurable select list column sizing

---

## v0.58.1 — 2026-03-14

**Bug fixes:**
- `coding-agent`: Sync tool hooks with agent event processing
- `coding-agent`: Fix session_directory hook lifecycle
- `ai`: Handle unknown `finish_reason` in openai-completions gracefully
- `tui`: Distinguish `Ctrl+Backspace` from `Backspace` on Windows Terminal
- `ai`: Replace curl with fetch in Anthropic OAuth token exchange

**Features:**
- `session-manager`: Allow supplying custom session ID in `newSession()`
- `tui`: Treat paste markers as atomic segments in editor
- `tui`: Support digit keybindings in session tree
- `coding-agent`: Add `session_directory` extension event and refinements

---

## v0.58.0 — 2026-03-14

**Features:**
- Raise context length to 1M tokens
- `coding-agent`: Add `session_directory` extension lifecycle event
- `tui`: Non-capturing overlay focus control

**Bug fixes:**
- `ai`: Send tool result images in `function_call_output`
- `coding-agent`: Keep only ISO date in system prompt
- `ai`: Improve Anthropic OAuth flow

---

## v0.57.1 — 2026-03-08

**Bug fixes:**
- `coding-agent`: Use shell for external editor on Windows
- `tui`: Chain slash-arg autocomplete after Tab completion
- `tui`: Handle tmux xterm extended keys; warn on tmux setup issues
- `tui`: Autocomplete highlight follows first prefix match as user types
- `coding-agent`: Custom tool collapsed/expanded rendering in HTML export
- `coding-agent`: Use strict JSONL framing
- `coding-agent`: Keep `~/.agents` skills user-scoped

---

## v0.57.0 — 2026-03-07

**Features:**
- `coding-agent`: Provider payload hook (`onPayload` / `before_provider_request` extension event)
- `tui`: Non-capturing overlays with focus control
- `coding-agent`: Preserve custom editor `onEscape`/`onCtrlD` handlers
- `ai`: Add `claude-sonnet-4-6` to Antigravity provider
- `coding-agent`: Add `treeFilterMode` setting for `/tree` default filter
- `ai`: Add `github-copilot gpt-5.3-codex` fallback model
- `coding-agent`: Add `session_directory` extension hook
- `tui`: Support digit keybindings

---

## v0.56.3 — 2026-03-06

**Bug fixes:**
- `coding-agent`: Guard against stale pre-compaction usage in error-path threshold check
- `coding-agent`: Truncate tool results in compaction summarization to prevent overflow
- `coding-agent`: Allow threshold compaction for error messages using last successful usage
- `coding-agent`: Retry sync lockfile acquisition to prevent false auth errors during parallel startup
- `tui`: Add `modifyOtherKeys` fallback for modified Enter keys in tmux
- `coding-agent`: Preserve thinking defaults across model switches
- `coding-agent`: Clear header on `/new`

---

## v0.56.2 — 2026-03-06

**Bug fixes:**
- `ai`: Restore OpenAI Responses reasoning replay
- `tui`: Add Kitty CSI-u printable decoding to Input component
- `ai`: Keep Mistral browser-safe and preserve Mistral thinking replay
- `ai`: Cap GPT-5.4 context windows to 272k
- `coding-agent`: Normalize CRLF in write preview rendering

---

## v0.56.1 — 2026-03-05

**Bug fixes:**
- `ai`: Handle redacted thinking blocks; skip interleaved beta for adaptive models; drop temperature with thinking
- `ai`: Use `enable_thinking` for Z.ai instead of `thinking` param
- `coding-agent`: Finalize provider unregister lifecycle; dependency security updates
- `coding-agent`: Ignore SIGINT while process is suspended

---

## v0.56.0 — 2026-03-04

**Features:**
- `ai`: Add OpenCode Go provider support
- `ai`: Add `gemini-3.1-pro-preview` to google-gemini-cli provider
- `coding-agent`: Flush `registerProvider` immediately after `bindCore`; add `unregisterProvider`
- `coding-agent`: Add `branchSummary.skipPrompt` setting

**Bug fixes:**
- `ai`: Enable adaptive thinking for Claude Sonnet 4.6 and clamp xhigh effort
- `ai`: Vertex ADC credentials async import race fix
- Various subagent and user-agent path fixes

---

## v0.55.4 — 2026-03-02

**Bug fixes:**
- `coding-agent`: Close retry wait race across queued events
- `coding-agent`: Add tool `promptGuidelines` support
- `coding-agent`: Support dynamic tool registration and tool prompt snippets (v0.55.4+)
- `coding-agent`: Serialize session event handling to preserve message order
- `coding-agent`: Remove extra spacer before streaming tool blocks
- `coding-agent`: Allow suppressing custom tool transcript blocks
- `coding-agent`: Use Alt+V for image pasting on Windows

---

## v0.55.3 — 2026-02-27

**Bug fixes:**
- `coding-agent`: Prevent duplicate session headers when forking from pre-assistant entry

---

## v0.55.2 — 2026-02-27

**Bug fixes:**
- `mom`: Fix settings manager API drift crash via `SettingsManager`

---

## v0.55.1 — 2026-02-26

**Bug fixes:**
- `ai`: Fix `enable_thinking` Z.ai parameter
- Various stability fixes

---

## v0.55.0 — 2026-02-24

**Features:**
- **Resource precedence**: Project resources (`.pi/`) now take precedence over user-global (`~/.pi/agent/`). Extension resource conflicts logged instead of unloading extensions.

**Bug fixes:**
- `tui`: Kitty CSI-u decode improvements

---

## v0.54.2 — 2026-02-23

**Bug fixes:**
- `coding-agent`: `AuthStorage` constructor breaking change documentation
- Various fixes

---

## v0.54.1 — 2026-02-22

**Bug fixes:**
- `coding-agent`: Support dynamic tool registration (v0.55.4 backport preview)

---

## v0.54.0 — 2026-02-20

**Features:**
- `coding-agent`: Discover skills in `.agents/skills/` paths by default (shared with other agents)
- `ai`: Add Claude Sonnet 4.6 model fallback
- `tui`: Enable VT input mode on Windows

---

## v0.53.1 — 2026-02-19

**Bug fixes:**
- Model provider summary corrections

---

## v0.53.0 — 2026-02-17

**Features:**
- `ai`: Add `gpt-5.3-codex-spark` model definition
- `ai`: Route GitHub Copilot Claude via Anthropic Messages API

**Bug fixes:**
- `coding-agent`: Tighten git source parsing and local path normalization
- `tui`: Scope `@` fuzzy search to path prefixes
- `ai`: Tolerate malformed trailing tool-call JSON in OpenAI streams
- `web-ui`: Make model selector search case-insensitive
- `coding-agent`: Clear extension terminal input listeners on reset
- `coding-agent`: Honor `--model` selection, thinking, and `--api-key` flags

---

## v0.52.12 — 2026-02-13

**Features:**
- Configurable transport options; Codex WebSocket session caching

**Bug fixes:**
- `coding-agent`: Show unknown context usage after compaction; fix multi-compaction boundary
- `extensions`: Forward message and tool execution events to extensions
- Terminal input hook for extensions

---

## v0.52.0 — 2026-02-05

**Features:**
- Extension message and tool execution event forwarding
- Terminal input hook for extensions (`terminal_input` event)

---

## v0.51.4 — 2026-02-03
- `coding-agent`: Share URLs default to **pi.dev** (donated by exe.dev)

---

## v0.51.0 — 2026-02-01 *(Breaking)*
- **Breaking** `coding-agent`: Extension `ToolDefinition.execute` parameter order changed
- `coding-agent`: Android/Termux support; Linux ARM64 musl prebuilds
- `coding-agent`: Bash spawn hook for extensions; Nix/Guix support via `PI_PACKAGE_DIR`
- `coding-agent`: Named-session filter in `/resume`

---

## v0.50.0 — 2026-01-26 *(Breaking — Pi packages)*
- **Breaking** `coding-agent`: External packages now configured via `packages` array, not `extensions`
- `coding-agent`: **Pi package system** — bundle and install extensions, skills, prompts, themes via `pi install/remove/update/list/config`
- `coding-agent`: Hot reload of all resources (`/reload`)
- `coding-agent`: Custom providers via `pi.registerProvider()`
- `ai`: Azure OpenAI Responses provider; OpenRouter routing controls
- `ai`: Hugging Face provider; Vercel AI Gateway provider

---

## v0.49.0–v0.49.3 — 2026-01-17 to 2026-01-22
- `coding-agent`: Emacs-style kill-ring editing
- `coding-agent`: `ctx.compact()` and `ctx.getContextUsage()` exposed to extensions
- `web-ui`: tsgo bumped to 7.0.0-dev.20260120.1 for decorator support

---

## v0.48.0 — 2026-01-16
- `coding-agent`: `quietStartup`, `shellCommandPrefix`, `editorPaddingX` settings
- `coding-agent`: Extension command argument autocompletions

---

## v0.47.0 — 2026-01-16 *(Breaking)*
- **Breaking** `tui`: `Editor` constructor now requires `TUI` as first parameter
- `coding-agent`: **OpenAI Codex official support** with full compatibility
- `coding-agent`: IME / hardware-cursor support, `input` extension event
- `coding-agent`: `pi-internal://` URL scheme for the read tool

---

## v0.46.0 — 2026-01-15
- `coding-agent`: Edit tool fuzzy-match fallback; `APPEND_SYSTEM.md` support
- `ai`: MiniMax China provider
- `tui`: Kitty keyboard layout fixes for non-Latin layouts

---

## v0.45.0 — 2026-01-13
- `ai`: MiniMax provider; Amazon Bedrock provider (experimental)
- `ai`: Vercel AI Gateway provider; `serviceTier`; OpenRouter Anthropic caching
- `coding-agent`: Share URLs migrated from `shittycodingagent.ai` to `buildwithpi.ai`

---

## v0.44.0 — 2026-01-12 *(Breaking)*
- **Breaking** `coding-agent`: `pi.getAllTools()` returns `ToolInfo[]` instead of `string[]`
- `coding-agent`: Session naming via `/name` command

---

## v0.42.0 — 2026-01-09
- `ai` / `coding-agent`: OpenCode Zen provider support
- `coding-agent`: Anthropic OAuth support restored

---

## v0.39.0 — 2026-01-08 *(Breaking)*
- **Breaking** `coding-agent`: `before_agent_start` event returns `systemPrompt` instead of `systemPromptAppend`
- `coding-agent`: Pluggable operations for built-in tools (remote execution)
- `tui`: Experimental overlay compositing for `ctx.ui.custom()`

---

## v0.38.0 — 2026-01-08 *(Breaking)*
- **Breaking** `coding-agent`: `ctx.ui.custom()` factory signature changed
- `coding-agent`: `--no-extensions` flag; SDK exports; `thinkingBudgets` setting
- `ai`: Removed OpenAI Codex per-thinking-level model aliases

---

## v0.37.0 — 2026-01-05 *(Breaking)*
- **Breaking** `ai`: OpenAI Codex models no longer have per-thinking-level variants
- `coding-agent`: Headless OAuth support for all providers

---

## v0.36.0 — 2026-01-05
- `ai` / `coding-agent`: **OpenAI Codex OAuth provider** (experimental)

---

## v0.35.0 — 2026-01-05 *(Breaking — Extensions unification)*
- **Breaking** `coding-agent`: Hooks and custom tools unified into single **extensions** system
- **Breaking** `coding-agent`: "Slash commands" renamed to **prompt templates**
- **Breaking** `coding-agent`: `HookAPI` → `ExtensionAPI`, `CustomTool` → `ToolDefinition`
- `coding-agent`: Auto-migration of `hooks/`, `tools/` → `extensions/`; `commands/` → `prompts/`
- `coding-agent`: `--hook` / `--tool` CLI flags → `--extension`

---

## v0.34.0 — 2026-01-04
- `coding-agent`: Hook API additions: `setActiveTools`, `registerFlag`, `registerShortcut`, `setWidget`
- `coding-agent`: Plan-mode example hook
- `tui`: `SymbolKey`, `getExpandedText`

---

## v0.33.0 — 2026-01-04 *(Breaking)*
- **Breaking** `tui`: All `isXxx()` key-detection functions removed, replaced with `matchesKey()`
- `coding-agent`: Clipboard image paste via Ctrl+V
- `coding-agent`: Configurable keybindings via `keybindings.json`
- `coding-agent`: `/quit` and `/exit` aliases

---

## v0.32.0 — 2026-01-03 *(Breaking)*
- **Breaking** `agent`: Queue API replaced with `steer()` / `followUp()` (different delivery semantics)
- `ai`: Vertex AI provider support
- `coding-agent`: `!!command` for shell commands excluded from LLM context
- `coding-agent`: Image auto-resize

---

## v0.31.0 — 2026-01-02 *(Major architectural release — Session trees)*
- **Breaking** `ai`: Agent API moved to `@earendil-works/pi-agent-core`
- **Breaking** `agent-core`: Transport abstraction removed; `AppMessage` renamed to `AgentMessage`
- **Breaking** `web-ui`: Agent class moved to `@earendil-works/pi-agent-core`
- **Breaking** `agent`: Queue API replaced with `steer()` / `followUp()`
- `coding-agent`: **Session trees** — tree structure with `id`/`parentId`, in-place branching via `/tree`
- `coding-agent`: Structured compaction with file tracking
- `coding-agent`: `/share` command for shareable session URLs via `shittycodingagent.ai`
- `coding-agent`: RPC protocol migration

---

## v0.29.0 — 2025-12-25 *(Breaking)*
- **Breaking** `coding-agent`: Renamed `/clear` to `/new`
- `coding-agent`: Unified `/settings` command; `SYSTEM.md` auto-loading

---

## v0.28.0 — 2025-12-25 *(Breaking)*
- **Breaking** `ai`: OAuth storage removed — callers now manage credentials
- `coding-agent`: Credential storage refactored to `auth.json`
- `coding-agent`: `AuthStorage` and `ModelRegistry` classes

---

## v0.26.0 — 2025-12-22
- `coding-agent`: **SDK for programmatic usage** — `createAgentSession()` factory
- `coding-agent`: Project-specific settings (`.pi/settings.json`)

---

## v0.25.0 — 2025-12-20
- `ai` / `coding-agent`: **Google Gemini CLI OAuth provider** (free Gemini access)
- `ai` / `coding-agent`: **Google Antigravity OAuth provider** (free Gemini 3, Claude, GPT access)
- `coding-agent`: Interruptible tool execution

---

## v0.24.0 — 2025-12-19 *(Breaking)*
- **Breaking** `coding-agent`: Custom tools require an `index.ts` entry point
- `coding-agent`: Subagent orchestration example
- `tui`: Kitty keyboard protocol support
- `coding-agent`: Dynamic API-key refresh for OAuth tokens

---

## v0.23.4 — 2025-12-18
- `coding-agent`: Syntax highlighting for code blocks, read tool, and write tool output

---

## v0.23.0 — 2025-12-17 *(Breaking)*
- **Breaking** `coding-agent`: Replaced `session_start` / `session_switch` hooks with unified `session` event
- `coding-agent`: **Custom tools** — extend pi with TypeScript tools having custom TUI rendering

---

## v0.22.3 — 2025-12-16
- `coding-agent`: **Streaming bash output** in real time during execution
- `agent`: Tool result streaming (`tool_execution_update` event)

---

## v0.22.0 — 2025-12-15
- `ai` / `coding-agent`: **GitHub Copilot provider** via OAuth login
- `ai`: Interleaved thinking for Anthropic Claude 4 models

---

## v0.21.0 — 2025-12-14
- `coding-agent`: **Inline image rendering** via Kitty graphics protocol and iTerm2
- `ai`: Gemini 3 Pro thinking support

---

## v0.20.0 — 2025-12-13 *(Breaking)*
- **Breaking** `coding-agent`: Skills must use `SKILL.md` convention in directories

---

## v0.19.0 — 2025-12-12
- `coding-agent`: **Skills system** — auto-discover instruction files from Claude Code, Codex CLI, and Pi-native paths

---

## v0.18.2 — 2025-12-11
- `coding-agent`: Auto-retry on transient errors (429, 5xx) with exponential backoff

---

## v0.18.1 — 2025-12-10
- `ai` / `coding-agent`: **Mistral provider** support

---

## v0.18.0 — 2025-12-10
- `coding-agent`: **Hooks system** — TypeScript modules extending agent behavior via lifecycle events (`tool_call`, `tool_result`, `session_start`, etc.)
- `coding-agent`: `pi.send()` API for injecting messages; `--hook` CLI flag
- `coding-agent`: Hook UI primitives: `ctx.ui.select()`, `confirm()`, `input()`, `notify()`

---

## v0.17.0 — 2025-12-09 *(Breaking)*
- **Breaking** `ai`: Removed provider-level tool-argument validation
- `coding-agent`: Simplified compaction flow
- `agent`: `agentLoopContinue` function

---

## v0.16.0 — 2025-12-09 *(Breaking)*
- **Breaking** `coding-agent`: Completely redesigned RPC protocol with new JSON framing

---

## v0.15.0 — 2025-12-09
- `coding-agent`: Major code refactoring into `core/`, `modes/`, `utils/`, `cli/` directories

---

## v0.14.0 — 2025-12-08
- `ai`: `xhigh` thinking level for OpenAI codex-max models
- `coding-agent`: Bash mode — execute shell commands with `!` prefix directly from editor
- `coding-agent`: OpenAI compatibility overrides in `models.json`

---

## v0.13.0 — 2025-12-06 *(Breaking)*
- **Breaking** `ai`: Added `totalTokens` to `Usage` type
- `coding-agent`: Windows shell configuration for bash tool

---

## v0.12.10 — 2025-12-04
- `ai` / `coding-agent`: `gpt-5.1-codex-max` model support
- `ai`: Fixed OpenAI token counting; Claude Opus 4.5 cache pricing

---

## v0.12.7 — 2025-12-04
- `coding-agent`: **Context compaction** — `/compact`, `/autocompact`, automatic compaction at threshold
- `coding-agent`: HTML export support for compaction

---

## v0.12.0 — 2025-12-02
- `coding-agent`: Print mode (`-p`, `-P`); `--print-turn` for multi-turn
- `coding-agent`: `--no-markdown`; auto-save in print mode; thinking level CLI flags

---

## v0.11.0 — 2025-11-29
- `coding-agent`: File-based slash commands
- `coding-agent`: `/branch` command for conversation branching
- `coding-agent`: Drag-and-drop files; `@path` content references
- `coding-agent`: Model selector with search

---

## v0.10.2 — 2025-11-26
- `coding-agent`: Thinking-level persistence; model cycling (Ctrl+I)
- `coding-agent`: Automatic retry with backoff; token usage/cost in footer
- `coding-agent`: `--system-prompt` flag

---

## v0.10.1 — 2025-11-25
- `coding-agent`: Custom model configuration via `~/.pi/models.json`

---

## v0.10.0 — 2025-11-25 *(Initial public release)*
- First public npm release of `@mariozechner/pi-coding-agent`
- `coding-agent`: Interactive TUI with streaming responses
- `coding-agent`: Session management; tools (`read`, `write`, `edit`, `bash`, `glob`, `grep`, `think`)
- `coding-agent`: Thinking mode for Claude
- `coding-agent`: `@` file autocompletion; `/export` HTML export; `/model` runtime switching
- `coding-agent`: Anthropic / OpenAI / Google provider support
- `ai`: Initial release with multi-provider LLM support

---

## Earlier Releases (Pre-v0.10.0)

The `v0.0.1` through `v0.9.4` tags represent pre-release development.

- `v0.9.4` (2025-11-26): Initial `pi-ai` release with Anthropic, OpenAI, Google provider support
- `v0.0.1` – `v0.5.x`: Internal prototyping

```bash
git tag --sort=-version:refname   # List all releases
git show v0.10.0                  # First public release
```

The pre-v0.10.0 history reflects rapid iteration before the monorepo structure was stabilized and the first public release was cut.

---

## Released vs Unreleased Boundary

**v0.74.0 (2026-05-07) is the cut.** Anything landing on `main` after that commit is unreleased and must not be documented as a shipped feature until a post-v0.74.0 tag exists. See [Era 9](./09-version-history.md#era-9-scope-migration-and-final-old-scope-releases-v0730--v0740-2026-05-04-to-2026-05-07) in the detailed version history for the in-flight items currently sitting on `main`.
