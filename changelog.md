# Changelog

All pi-mono releases follow lockstep versioning — all packages (`pi-ai`, `pi-tui`, `pi-agent-core`, `pi-coding-agent`, `pi-web-ui`, `pi-mom`, `pi-pods`) are released together at the same version.

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
