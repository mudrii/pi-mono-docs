# Changelog

All pi-mono releases follow lockstep versioning — all packages (`pi-ai`, `pi-tui`, `pi-agent-core`, `pi-coding-agent`, `pi-web-ui`, `pi-mom`, `pi-pods`) are released together at the same version.

---

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

## v0.51.0 — 2026-02-02

**Features:**
- Various provider and stability improvements

---

## v0.50.0 — 2026-01-26

**Features:**
- Extended context windows
- Provider stability improvements

---

## v0.49.0 — 2026-01-17

Initial public releases in the `0.49.x` series.

---

## v0.48.0 — 2026-01-17

First broadly available release. Established the core architecture:

- 7-package monorepo with lockstep versioning
- `pi-ai`: 23+ LLM providers, 9 wire protocols
- `pi-agent-core`: Stateful agent with parallel tool execution
- `pi-tui`: Differential terminal rendering, Kitty graphics protocol
- `pi-coding-agent`: Interactive coding CLI with 4 built-in tools
- `pi-web-ui`: Lit-based web components, IndexedDB storage
- `pi-mom`: Slack bot with self-managing capabilities
- `pi-pods`: GPU pod management for vLLM inference

---

## Earlier Releases

Releases prior to v0.48.0 are available in the git tag history:

```bash
git tag --sort=-version:refname   # List all releases
git show v0.7.22                  # View a specific release commit
```

Notable early versions: `v0.7.x`, `v0.6.0`, `v0.5.x`, `v0.0.1`, `v0.0.2`.

The repository and its early history (pre-v0.48) reflect rapid iteration before the monorepo structure was stabilized.
