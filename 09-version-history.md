# Pi-Mono Version History

## Overview

Pi-Mono uses **lockstep versioning** across all packages. Every release bumps all six packages to the same version number simultaneously, even if a particular package has no changes in that release. This means `@mariozechner/pi-coding-agent`, `@mariozechner/pi-ai`, `@mariozechner/pi-tui`, `@mariozechner/pi-agent-core` (formerly `@mariozechner/pi-agent`), `@mariozechner/pi-mom`, and `@mariozechner/pi-web-ui` always share the same version.

As of 2026-04-09, the project has **241 release tags** spanning from `v0.0.1` to `v0.65.2`, with the first public release (v0.10.0) on 2025-11-25 and the latest (v0.65.2) on 2026-04-06 -- roughly 4 months of rapid development.

---

## Era 1: Genesis (v0.0.1 -- v0.9.4, up to 2025-11-26)

These early tags represent pre-release development. The `pi-ai` package had its initial release at v0.9.4 with multi-provider LLM support. The `pi-mom` Slack bot also debuted at v0.9.4 with basic Slack integration, Docker sandbox mode, bash/read/write/edit tools, and thread-based output.

### Key milestones
- v0.0.1 -- v0.5.x: Internal prototyping
- v0.9.4 (2025-11-26): Initial `pi-ai` release with Anthropic, OpenAI, Google provider support; initial `pi-mom` Slack bot release

---

## Era 2: Public Launch & Foundation (v0.10.0 -- v0.12.x, 2025-11-25 to 2025-12-06)

### v0.10.0 (2025-11-25) -- Initial public release
- **coding-agent**: Interactive TUI with streaming responses, session management, tools (read, write, edit, bash, glob, grep, think), thinking mode for Claude, `@` file autocompletion, `/export` HTML export, `/model` runtime switching, Anthropic/OpenAI/Google provider support
- First public npm release of `@mariozechner/pi-coding-agent`

### v0.10.1 (2025-11-25)
- Custom model configuration via `~/.pi/models.json`

### v0.10.2 (2025-11-26)
- Thinking level persistence, model cycling (Ctrl+I), automatic retry with backoff, token usage/cost in footer, `--system-prompt` flag

### v0.11.0 (2025-11-29)
- File-based slash commands, `/branch` command for conversation branching, drag & drop files, `@path` content references, model selector with search

### v0.12.0 (2025-12-02)
- Print mode (`-p`, `-P`), `--print-turn` for multi-turn, `--no-markdown`, auto-save in print mode, thinking level CLI flags

### v0.12.7 (2025-12-04)
- **Context compaction**: `/compact`, `/autocompact`, automatic compaction at threshold, HTML export support

### v0.12.10 (2025-12-04)
- `gpt-5.1-codex-max` model support
- **pi-ai**: Fixed OpenAI token counting, Claude Opus 4.5 cache pricing

---

## Era 3: Extensibility & Provider Expansion (v0.13.0 -- v0.22.x, 2025-12-06 to 2025-12-16)

### v0.13.0 (2025-12-06)
- **Breaking**: Added `totalTokens` to `Usage` type in pi-ai
- Windows shell configuration for bash tool

### v0.14.0 (2025-12-08)
- **xhigh thinking level** for OpenAI codex-max models
- **Bash mode**: Execute shell commands with `!` prefix directly from editor
- OpenAI compatibility overrides in `models.json`

### v0.15.0 (2025-12-09)
- Major code refactoring into `core/`, `modes/`, `utils/`, `cli/`

### v0.16.0 (2025-12-09)
- **Breaking**: Completely redesigned RPC protocol with new JSON protocol

### v0.17.0 (2025-12-09)
- Simplified compaction flow, `agentLoopContinue` function in pi-ai
- **Breaking**: Removed provider-level tool argument validation

### v0.18.0 (2025-12-10)
- **Hooks system**: TypeScript modules extending agent behavior via lifecycle events (tool_call, tool_result, session_start, etc.)
- `pi.send()` API for injecting messages, `--hook` CLI flag
- Hook UI primitives: `ctx.ui.select()`, `confirm()`, `input()`, `notify()`

### v0.18.1 (2025-12-10)
- **Mistral provider** support

### v0.18.2 (2025-12-11)
- **Auto-retry on transient errors** (429, 5xx) with exponential backoff

### v0.19.0 (2025-12-12)
- **Skills system**: Auto-discover instruction files from Claude Code, Codex CLI, and Pi-native paths

### v0.20.0 (2025-12-13)
- **Breaking**: Skills must use `SKILL.md` convention in directories

### v0.21.0 (2025-12-14)
- **Inline image rendering** via Kitty graphics protocol and iTerm2

### v0.22.0 (2025-12-15)
- **GitHub Copilot provider** support via OAuth login
- Interleaved thinking for Anthropic Claude 4 models

### v0.22.3 (2025-12-16)
- **Streaming bash output** in real-time during execution
- Tool result streaming (`tool_execution_update` event)

---

## Era 4: Session Trees & API Redesign (v0.23.0 -- v0.31.x, 2025-12-17 to 2026-01-02)

### v0.23.0 (2025-12-17)
- **Custom tools**: Extend pi with TypeScript tools that have custom TUI rendering and user interaction
- **Breaking**: Replaced `session_start`/`session_switch` hooks with unified `session` event

### v0.23.4 (2025-12-18)
- **Syntax highlighting** for code blocks, read tool, write tool output

### v0.24.0 (2025-12-19)
- Subagent orchestration example, Kitty keyboard protocol support
- Dynamic API key refresh for OAuth tokens
- **Breaking**: Custom tools require `index.ts` entry point

### v0.25.0 (2025-12-20)
- **Google Gemini CLI OAuth provider** (free Gemini access)
- **Google Antigravity OAuth provider** (free Gemini 3, Claude, GPT access)
- Interruptible tool execution

### v0.26.0 (2025-12-22)
- **SDK for programmatic usage**: `createAgentSession()` factory
- Project-specific settings (`.pi/settings.json`)

### v0.28.0 (2025-12-25)
- **Breaking (pi-ai)**: OAuth storage removed, callers manage credentials
- Credential storage refactored to `auth.json`
- `AuthStorage` and `ModelRegistry` classes

### v0.29.0 (2025-12-25)
- **Breaking**: Renamed `/clear` to `/new`
- Unified `/settings` command, `SYSTEM.md` auto-loading

### v0.31.0 (2026-01-02) -- Major architectural release
- **Session trees**: Tree structure with `id`/`parentId`, in-place branching via `/tree`
- **Breaking (pi-ai)**: Agent API moved to `@mariozechner/pi-agent-core`
- **Breaking (agent-core)**: Transport abstraction removed, `AppMessage` renamed to `AgentMessage`
- **Breaking (web-ui)**: Agent class moved to `@mariozechner/pi-agent-core`
- **Breaking (mom)**: Multiple type and API renames
- **Breaking (agent)**: Queue API replaced with `steer()`/`followUp()`
- Structured compaction with file tracking
- `/share` command for shareable session URLs via shittycodingagent.ai

---

## Era 5: Extensions Unification & Package System (v0.32.0 -- v0.50.x, 2026-01-03 to 2026-01-26)

### v0.32.0 (2026-01-03)
- **Breaking**: Queue API replaced with `steer()`/`followUp()` with different delivery semantics
- Vertex AI provider support
- `!!command` for shell commands excluded from LLM context

### v0.33.0 (2026-01-04)
- **Breaking (tui)**: All `isXxx()` key detection functions removed, replaced with `matchesKey()`
- Clipboard image paste via Ctrl+V
- Configurable keybindings via `keybindings.json`

### v0.35.0 (2026-01-05) -- Extensions unification
- Hooks and custom tools unified into single **extensions** system
- "Slash commands" renamed to **prompt templates**
- `HookAPI` -> `ExtensionAPI`, `CustomTool` -> `ToolDefinition`
- Auto-migration of `hooks/`, `tools/` -> `extensions/`; `commands/` -> `prompts/`

### v0.36.0 (2026-01-05)
- **OpenAI Codex OAuth provider** support

### v0.37.0 (2026-01-05)
- **Breaking (pi-ai)**: OpenAI Codex models no longer have per-thinking-level variants
- Headless OAuth support for all providers

### v0.38.0 (2026-01-08)
- **Breaking**: `ctx.ui.custom()` factory signature changed
- `--no-extensions` flag, SDK exports, `thinkingBudgets` setting

### v0.39.0 (2026-01-08)
- **Breaking**: `before_agent_start` event returns `systemPrompt` instead of `systemPromptAppend`
- Pluggable operations for built-in tools (remote execution)
- Experimental overlay compositing for `ctx.ui.custom()`

### v0.42.0 (2026-01-09)
- OpenCode Zen provider support
- Anthropic OAuth support restored

### v0.44.0 (2026-01-12)
- **Breaking**: `pi.getAllTools()` returns `ToolInfo[]` instead of `string[]`
- Session naming (`/name` command)

### v0.45.0 (2026-01-13)
- MiniMax provider, Amazon Bedrock provider (experimental)
- Vercel AI Gateway provider
- Share URLs changed from shittycodingagent.ai to buildwithpi.ai

### v0.45.4 (2026-01-13)
- Replaced `sharp` with `wasm-vips` for image processing (eliminates native build issues)

### v0.46.0 (2026-01-15)
- Edit tool fuzzy matching fallback, `APPEND_SYSTEM.md` support
- MiniMax China provider, Kitty keyboard layout fixes for non-Latin layouts

### v0.47.0 (2026-01-16)
- **Breaking (tui)**: `Editor` constructor requires `TUI` as first parameter
- **OpenAI Codex official support** with full compatibility
- IME/hardware cursor support, `input` extension event
- `pi-internal://` URL scheme for read tool

### v0.48.0 (2026-01-16)
- `quietStartup` setting, `shellCommandPrefix` setting, `editorPaddingX`
- Extension command argument auto-completions

### v0.49.0 (2026-01-17)
- Emacs-style kill ring editing, `ctx.compact()` and `ctx.getContextUsage()` for extensions

### v0.50.0 (2026-01-26) -- Package system
- **Pi packages** for bundling and installing extensions, skills, prompts, themes
- **Hot reload** (`/reload`) of all resources
- **Custom providers** via `pi.registerProvider()`
- Azure OpenAI Responses provider, OpenRouter routing
- `pi install`, `pi remove`, `pi update`, `pi list`, `pi config` commands
- **Breaking**: External packages configured via `packages` array instead of `extensions`

---

## Era 6: Model Evolution & Platform Maturity (v0.51.0 -- v0.60.0, 2026-02-01 to 2026-03-18)

### v0.51.0 (2026-02-01)
- **Breaking**: Extension `ToolDefinition.execute` parameter order changed
- Android/Termux support, Linux ARM64 musl support
- Bash spawn hook for extensions, Nix/Guix support (`PI_PACKAGE_DIR`)
- Named session filter in `/resume`

### v0.51.4 (2026-02-03)
- Share URLs default to **pi.dev** (donated by exe.dev)

### v0.52.0 (2026-02-05) -- Claude Opus 4.6 era
- **Claude Opus 4.6** model support
- **GPT-5.3 Codex** model support
- SSH URL support for git packages, `auth.json` shell command resolution
- Default model updated to Claude Opus 4.6 for Anthropic provider

### v0.52.7 (2026-02-06)
- Per-model overrides in `models.json` via `modelOverrides`
- Bedrock proxy support (`AWS_BEDROCK_SKIP_AUTH`)

### v0.52.10 (2026-02-12)
- Extension terminal input interception (`terminal_input`)
- Expanded CLI `--model` with `provider/id` syntax and fuzzy matching
- **Breaking**: `ContextUsage.tokens` and `ContextUsage.percent` now `number | null`
- GLM-5 model support

### v0.52.12 (2026-02-13)
- Transport setting (`sse`/`websocket`/`auto`) for providers like OpenAI Codex
- MiniMax M2.5 model entries

### v0.53.0 (2026-02-17)
- **Breaking**: `SettingsManager` persistence semantics changed (async disk writes)
- **Breaking**: `AuthStorage` constructor no longer public
- Claude Sonnet 4.6 model support

### v0.54.0 (2026-02-19)
- Auto-discovery of `.agents/skills` directories

### v0.55.0 (2026-02-24)
- **Breaking**: Resource precedence changed to project-first (`cwd/.pi`) before user-global (`~/.pi/agent`)
- Extension registration conflicts resolved by first registration in load order

### v0.55.1 (2026-02-26)
- Offline startup mode (`--offline` / `PI_OFFLINE`)
- `gemini-3.1-pro-preview` model support

### v0.55.2 (2026-02-27)
- Dynamic provider unregistration via `pi.unregisterProvider()`

### v0.55.4 (2026-03-02)
- Runtime tool registration applies immediately without `/reload`
- Tool `promptSnippet` and `promptGuidelines` for system prompt customization

### v0.56.0 (2026-03-04)
- **Breaking (pi-ai)**: OAuth exports moved to `@mariozechner/pi-ai/oauth` subpath
- OpenCode Go provider, `branchSummary.skipPrompt` setting
- `gemini-3.1-flash-lite-preview` model support

### v0.56.2 (2026-03-05)
- **GPT-5.4** model support (default for OpenAI and OpenAI Codex)
- Mistral native SDK integration
- `treeFilterMode` setting

### v0.56.3 (2026-03-06)
- `claude-sonnet-4-6` via Google Antigravity
- tmux Shift+Enter/Ctrl+Enter via xterm modifyOtherKeys fallback

### v0.57.0 (2026-03-07)
- **Breaking**: RPC mode uses strict LF-only JSONL framing
- `before_provider_request` extension hook
- Non-capturing overlay focus control for extensions

### v0.57.1 (2026-03-07)
- Tree branch folding and segment-jump navigation in `/tree`
- `session_directory` extension event
- Digit keybindings (0-9) in keybinding system

### v0.58.0 (2026-03-14)
- **Claude Opus 4.6 / Sonnet 4.6 context window raised to 1M tokens** (from 200K)
- Extension tool calls execute in parallel by default
- `GOOGLE_CLOUD_API_KEY` for Vertex provider
- `beforeToolCall`/`afterToolCall` hooks in agent-core

### v0.58.1 (2026-03-14)
- `pi uninstall` alias, Qwen chat template compat mode
- Many Windows and WSL fixes

### v0.59.0 (2026-03-17)
- **Faster startup** by lazy-loading provider SDKs on first use
- Better provider retry behavior
- OSC 133 terminal integration markers
- **Breaking**: Custom tool `promptSnippet` required for system prompt inclusion

### v0.60.0 (2026-03-18) -- Current release
- **Session forking** from CLI with `--fork <path|id>`
- `createLocalBashOperations()` export for extensions
- **Breaking**: Startup no longer auto-updates unpinned packages; use `pi update` explicitly
- Background update checking with notifications in interactive mode
- Bedrock Claude 4.6 context window corrected to 200K
- OAuth callback handling aligned across all providers

---

## Era 7: Session Runtime & API Hardening (v0.61.0 – v0.65.2, 2026-03-20 to 2026-04-06)

This era focused on SDK ergonomics, API hardening, and reliability fixes. It introduced `AgentSessionRuntime` for safe SDK-level session switching, unified extension lifecycle events, and overhauled the `AgentState` API to be immutable by callers. Three releases contain breaking changes (v0.63.0, v0.64.0, v0.65.0).

### v0.61.0 (2026-03-20)
- **ai**: Added `gpt-5.4-mini` for `openai-codex` provider; `validateToolArguments` falls back gracefully in restricted runtimes (Cloudflare Workers); `google-vertex` API key placeholder fix; OpenRouter `reasoning.effort` payload fix; Bedrock prompt caching for inference profiles via `AWS_BEDROCK_FORCE_CACHE=1`

### v0.61.1 (2026-03-20)
- **ai**: Normalized MiniMax model metadata; added `MiniMax-M2.1-highspeed` entries for `minimax` and `minimax-cn` providers
- **tui**: Fixed shared keybinding resolution (user overrides no longer evict unrelated default shortcuts); fixed Termux software keyboard height changes causing full-screen redraws

### v0.62.0 (2026-03-23)
- **ai**: Added `BedrockOptions.requestMetadata` for AWS cost allocation tagging (forwards to Bedrock Converse API and appears in AWS Cost Explorer); exported `BedrockOptions` type; fixed OpenAI Responses foreign tool-call ID replay; fixed Anthropic thinking-disable handling; fixed explicit thinking disable across Google, Vertex, Gemini CLI, OpenAI Responses, OpenRouter
- **tui**: Fixed `truncateToWidth()` for very large strings; fixed markdown heading styling after inline code

### v0.63.0 (2026-03-27) **BREAKING**
- **ai**: Removed `ModelRegistry.getApiKey(model)` → use `getApiKeyAndHeaders(model)`; removed deprecated `minimax` and `minimax-cn` direct model IDs (use `MiniMax-M2.7` or `MiniMax-M2.7-highspeed`)
- **tui**: Added `PI_TUI_WRITE_LOG` directory for per-instance debug log files; fixed `@` autocomplete debouncing and cancellation; fixed viewport tracking after content shrinks; fixed blockquote inline element styling; fixed slash-command Tab completion chaining

### v0.63.1 (2026-03-27)
- **ai**: Added `gemini-3.1-pro-preview-customtools` for `google-vertex`; Ollama `prompt too long` overflow detection; Anthropic HTTP 413 `request_too_large` overflow detection
- **coding-agent**: Fixed repeated compaction dropping kept messages; fixed interactive compaction UI; fixed skill discovery recursion stopping at `SKILL.md`; fixed edit tool diff rendering for multi-edit; fixed auto-compaction for Ollama; fixed built-in tool override rendering

### v0.63.2 (2026-03-29)
- **agent**: Added `Agent.signal` to expose active `AbortSignal` for the current turn
- **coding-agent**: Added `ctx.signal` to `ExtensionContext`; edit tool unified to `edits[]` schema only (single-edit `oldText`/`newText` removed)
- **tui**: Fixed TUI cell size CSI response parsing (bare `Escape` no longer swallowed); Kitty keyboard protocol keypad normalization (fixes iTerm2 numpad)

### v0.64.0 (2026-03-29) **BREAKING**
- **ai**: Added faux provider helpers for deterministic tests: `registerFauxProvider()`, `fauxAssistantMessage()`, `fauxText()`, `fauxThinking()`, `fauxToolCall()`
- **agent**: Added `AgentTool.prepareArguments` hook for pre-validation argument normalization
- **coding-agent**: `ModelRegistry` public constructor removed — use `ModelRegistry.create(authStorage)` or `ModelRegistry.inMemory(authStorage)`; added `ToolDefinition.prepareArguments`; built-in `edit` tool uses `prepareArguments` for legacy schema compatibility; added `ctx.ui.setHiddenThinkingLabel()` for customizing collapsed thinking block label
- **tui**: Fixed TUI cell size CSI response; Kitty keypad functional key normalization

### v0.65.0 (2026-04-03) **BREAKING**
- **agent**: `AgentState` reshaped: `streamMessage` → `streamingMessage`, `error` → `errorMessage`; `isStreaming`, `streamingMessage`, `pendingToolCalls`, `errorMessage` now readonly; `tools`/`messages` are now accessor properties; all `setXxx()` mutator methods removed (use direct property assignment); `subscribe()` listeners now async and receive `AbortSignal`; `agent_end` is now the final emitted event
- **coding-agent**: Added `AgentSessionRuntime` / `createAgentSessionRuntime()` for SDK-level session switching; added `defineTool()` helper with TypeScript type inference; label timestamps in `/tree` with `Shift+T` toggle; unified diagnostics model (`info`/`warning`/`error`); removed `session_switch` and `session_fork` extension events (use `session_start` with `event.reason`); removed `session_directory` from extension and settings APIs; unknown single-dash CLI flags now error
- **ai**: Z.ai tool streaming support; Bedrock throttling fix (no longer misidentified as context overflow); various provider streaming fixes

### v0.65.1 (2026-04-05)
- **coding-agent**: Fixed bash output line-truncation (full output persisted to temp file); RpcClient forwards subprocess stderr in real-time; theme file watcher handles async `fs.watch` errors; session cwd prompts interactive users / fails non-interactive when original cwd missing; resource collision precedence fix (project/user resources override package resources); `--mode json` preserved for piped stdin; git/npm extension path CLI resolution; stale `/exit` docs removed from help output
- **ai**: OpenRouter `cached_tokens` normalization; `cache_write_tokens` preserved in completions stream usage

### v0.65.2 (2026-04-06)
- **tui**: Render scheduling coalesced to 16ms frame budget under streaming load; `requestRender(true)` still renders immediately
- **coding-agent**: Earendil startup announcement with inline image rendering; interactive Anthropic subscription auth warning (third-party usage billed per token from extra usage)
- **ai**: Updated generated model catalog (`packages/ai/src/models.generated.ts`)

---

## Version Numbering Pattern

The project uses semantic-ish versioning within the 0.x range:

- **Minor bumps** (0.50 -> 0.51 -> 0.52 etc.) indicate new features or breaking changes
- **Patch bumps** (0.52.1 -> 0.52.2 etc.) indicate bug fixes
- Some minor versions have many patches (e.g., v0.52.x had 13 patches)
- The pace averages roughly one minor release every 2-3 days during active development
- All packages bump in lockstep regardless of individual changes

## Breaking Changes Summary

The most significant breaking changes occurred at:
- **v0.28.0**: OAuth storage API redesign
- **v0.31.0**: Session trees, agent-core extraction, massive API renames
- **v0.35.0**: Hooks + custom tools unified into extensions
- **v0.50.0**: Package system replacing direct extension paths
- **v0.51.0**: Tool execute parameter reorder
- **v0.52.10**: ContextUsage nullability
- **v0.53.0**: SettingsManager async persistence
- **v0.55.0**: Resource precedence reversal (project-first)
- **v0.56.0**: OAuth export path change
- **v0.57.0**: RPC JSONL framing
- **v0.60.0**: Startup package update behavior change
- **v0.63.0**: `ModelRegistry.getApiKey()` removal; deprecated `minimax`/`minimax-cn` model IDs removed
- **v0.64.0**: `ModelRegistry` public constructor removal
- **v0.65.0**: `AgentState` reshape (field renames, readonly enforcement, accessor properties); all `Agent.setXxx()` mutator methods removed; `session_switch`/`session_fork` extension events removed; `session_directory` removed; unknown single-dash CLI flags now error

## Notable Model Milestones

| Date | Version | Model Event |
|------|---------|-------------|
| 2025-12-02 | v0.12.1 | GPT-4.1, o3, o4-mini added |
| 2025-12-04 | v0.12.10 | GPT-5.1-codex-max support |
| 2025-12-15 | v0.22.0 | GitHub Copilot provider |
| 2025-12-20 | v0.25.0 | Gemini CLI + Antigravity OAuth |
| 2026-01-05 | v0.36.0 | OpenAI Codex OAuth provider |
| 2026-01-13 | v0.45.0 | MiniMax + Amazon Bedrock |
| 2026-01-16 | v0.47.0 | OpenAI Codex official support |
| 2026-02-05 | v0.52.0 | Claude Opus 4.6, GPT-5.3 Codex |
| 2026-03-05 | v0.56.2 | GPT-5.4 support |
| 2026-03-14 | v0.58.0 | Claude 4.6 1M context window |
| 2026-03-20 | v0.61.0 | `gpt-5.4-mini` for `openai-codex` |
| 2026-03-27 | v0.63.1 | `gemini-3.1-pro-preview-customtools` for `google-vertex` |
