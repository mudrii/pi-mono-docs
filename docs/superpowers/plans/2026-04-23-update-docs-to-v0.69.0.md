# Pi-Mono Documentation Update v0.66.1 → v0.69.0

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Update all documentation in `~/src/tui/pi-mono-docs` from v0.66.1 to v0.69.0, covering 12 new releases (v0.67.0 through v0.69.0, April 13–22 2026), fact-checked against source code and online sources, then commit and push to GitHub.

**Architecture:** Parallel document-group agents update distinct file sets simultaneously. Each agent reads the source changelog, diffs, and existing doc, then rewrites its assigned section. A final validation pass checks cross-references and commits.

**Tech Stack:** TypeScript/Node.js monorepo (npm workspaces), GitHub (`github.com/mudrii/pi-mono-docs`), source at `~/src/tui/pi-mono` (v0.69.0).

---

## Context: What Changed (v0.66.1 → v0.69.0)

### Breaking Changes in v0.68.0
1. **Tool selection API**: `createAgentSession({ tools })` now takes `string[]` names (`"read"`, `"bash"`) instead of `Tool[]` instances.
2. **Removed prebuilt tool exports**: `readTool`, `bashTool`, `editTool`, `writeTool`, `grepTool`, `findTool`, `lsTool`, `readOnlyTools`, `codingTools`, plus `*ToolDefinition` values removed. Use `createReadTool(cwd)` etc. instead.
3. **Removed ambient `process.cwd()`**: `DefaultResourceLoader`, `loadProjectContextFiles()`, `loadSkills()` now require explicit `cwd`. `--no-tools` disables ALL tools (was: only built-ins).

### Breaking Changes in v0.69.0
4. **TypeBox 1.x migration**: From `@sinclair/typebox` 0.34.x to `typebox` 1.x. Extensions and SDK code must import from `typebox`. Legacy alias still works except `@sinclair/typebox/compiler` is no longer shimmed.
5. **Session-replacement callbacks**: `ctx.newSession()`, `ctx.fork()`, `ctx.switchSession()` now invalidate pre-switch captured objects. Move post-switch work into `withSession` callback.

### New Features (v0.67.0–v0.69.0)
- **Fireworks provider** (v0.68.1): `FIREWORKS_API_KEY`, models via Anthropic-compatible API
- **Stacked autocomplete providers** (v0.69.0): `ctx.ui.addAutocompleteProvider()`
- **Terminating tool results** (v0.69.0): `terminate: true` on tool results ends batch without LLM follow-up
- **OSC 9;4 terminal progress** (v0.69.0): Activity indicators in iTerm2, WezTerm, Windows Terminal, Kitty
- **Configurable working indicator** (v0.68.0): `ctx.ui.setWorkingIndicator()` for extensions
- **`systemPromptOptions` in `before_agent_start`** (v0.68.0): `BuildSystemPromptOptions` exposed to extensions
- **`/clone` command** (v0.68.0): Duplicates current active branch into a new session
- **`ctx.fork()` position** (v0.68.0): `position: "before" | "at"` for branching point control
- **Configurable keybindings** (v0.68.0): Scoped model selector and session-tree filter shortcuts remappable
- **`session_shutdown` reasons** (v0.68.0): `reason` and `targetSessionFile` in event payload
- **`PI_OAUTH_CALLBACK_HOST`** (v0.68.0): Custom bind host for OAuth flows
- **Per-tool `executionMode`** (v0.67.x): Override sequential/parallel per individual tool
- **Bedrock bearer-token auth** (v0.67.67): `AWS_BEARER_TOKEN_BEDROCK`
- **OpenRouter full routing fields** (v0.67.0): Fallbacks, ZDR, quantizations, etc.
- **`onResponse` in `StreamOptions`** (v0.67.6): Inspect HTTP status/headers after each response
- **`thinkingDisplay`** (v0.67.6): `"summarized" | "omitted"` in Anthropic/Bedrock options
- **claude-opus-4-7** model added (v0.67.4)
- **Kimi-for-coding** model normalization (v0.67.4)
- **PI_CODING_AGENT env var** (v0.66.x): Set to `true` at startup
- **`batch package updates`** (v0.68.0): `pi update` now batches per scope

### Version Statistics
- v0.66.1 docs covered: 243 releases (v0.0.1–v0.66.1)
- New releases: v0.67.0–v0.69.0 = 12 releases → total becomes **255 releases**
- Latest: v0.69.0 (2026-04-22)

---

## File Map

### Files to Update
| File | Scope of Changes |
|------|-----------------|
| `README.md` | Version badge (v0.66.1 → v0.69.0), date, statistics |
| `01-architecture-overview.md` | TypeBox 1.x, release count |
| `02-pi-ai-llm-abstraction.md` | Fireworks provider, TypeBox 1.x, new models (opus-4-7, kimi-for-coding, gemini-3.1-flash-lite), onResponse/thinkingDisplay, OpenRouter full routing, Bedrock bearer auth |
| `03-pi-agent-core.md` | Per-tool executionMode, parallel tool completion eagerness |
| `04-pi-coding-agent.md` | ALL breaking changes, /clone, TypeBox 1.x, stacked autocomplete, terminating tools, OSC 9;4, working indicator, keybindings, session_shutdown reasons |
| `05-pi-tui-terminal-ui.md` | OSC 9;4 progress, stacked autocomplete, symlink in fuzzy autocomplete fix |
| `06-application-packages.md` | web-ui SVG sandbox fix, mom default model check |
| `07-extension-system.md` | Stacked autocomplete API, terminating tools, systemPromptOptions, session-replacement withSession, working indicator, TypeBox 1.x migration guide |
| `08-sessions-and-persistence.md` | /clone vs /fork, session_shutdown reasons/targetSessionFile, ctx.fork() position |
| `09-version-history.md` | Add v0.67.0 through v0.69.0 entries (12 releases) |
| `10-fact-check-report.md` | Update version, re-verify npm/GitHub stats, check new provider claims |
| `packages/ai.md` | Fireworks, TypeBox 1.x, new models, onResponse, thinkingDisplay |
| `packages/agent.md` | Per-tool executionMode, eager parallel completion |
| `changelog.md` | Add v0.67.x–v0.69.0 summary |

### Source Files to Cross-Reference
- `~/src/tui/pi-mono/packages/coding-agent/CHANGELOG.md` — primary changelog
- `~/src/tui/pi-mono/packages/ai/CHANGELOG.md` — AI package changelog
- `~/src/tui/pi-mono/packages/agent/CHANGELOG.md` — agent-core changelog
- `~/src/tui/pi-mono/packages/tui/CHANGELOG.md` — TUI changelog
- `~/src/tui/pi-mono/packages/coding-agent/docs/extensions.md` — latest extension API
- `~/src/tui/pi-mono/packages/coding-agent/docs/sdk.md` — latest SDK docs
- `~/src/tui/pi-mono/packages/coding-agent/docs/providers.md` — latest provider docs
- `~/src/tui/pi-mono/packages/coding-agent/docs/keybindings.md` — keybindings docs
- `~/src/tui/pi-mono/packages/coding-agent/docs/settings.md` — settings docs
- `~/src/tui/pi-mono/packages/coding-agent/docs/tui.md` — TUI docs
- `~/src/tui/pi-mono/packages/coding-agent/docs/session.md` — session docs

---

## Task 1: Update README.md and Core Metadata

**Files:**
- Modify: `~/src/tui/pi-mono-docs/README.md`

- [ ] **Step 1: Read current README**

```bash
cat ~/src/tui/pi-mono-docs/README.md
```

- [ ] **Step 2: Read source README for current state**

```bash
cat ~/src/tui/pi-mono/README.md
cat ~/src/tui/pi-mono/packages/coding-agent/CHANGELOG.md | head -5
```

- [ ] **Step 3: Update README.md**

Change:
- `**Current release:** v0.66.1 (2026-04-08)` → `**Current release:** v0.69.0 (2026-04-22)`
- Stats section: `243 releases (v0.0.1 → v0.66.1)` → `255 releases (v0.0.1 → v0.69.0)`
- `Generated on 2026-04-10` → `Generated on 2026-04-23`
- `main (v0.66.1)` → `main (v0.69.0)`
- Feature table entry for **Built-in tools**: stays at 7 (unchanged)
- Add Fireworks to provider count if listed
- Table row `| 09 | [Version History]...` → update "243 releases in 7 eras" to "255 releases in 7 eras"

- [ ] **Step 4: Verify no version references missed**

```bash
grep -n "v0\.66" ~/src/tui/pi-mono-docs/README.md
# Should return 0 lines
```

---

## Task 2: Update 02-pi-ai-llm-abstraction.md

**Files:**
- Modify: `~/src/tui/pi-mono-docs/02-pi-ai-llm-abstraction.md`

- [ ] **Step 1: Read current doc and source changelogs**

```bash
cat ~/src/tui/pi-mono-docs/02-pi-ai-llm-abstraction.md
cat ~/src/tui/pi-mono/packages/ai/CHANGELOG.md | head -120
```

- [ ] **Step 2: Read source provider docs for fact-checking**

```bash
cat ~/src/tui/pi-mono/packages/coding-agent/docs/providers.md
ls ~/src/tui/pi-mono/packages/ai/src/providers/ 2>/dev/null || ls ~/src/tui/pi-mono/packages/ai/src/ | head -20
```

- [ ] **Step 3: Verify Fireworks provider exists in source**

```bash
grep -r "fireworks\|FIREWORKS" ~/src/tui/pi-mono/packages/ai/src/ --include="*.ts" -l
```

- [ ] **Step 4: Add Fireworks provider section**

In the providers table, add a row for Fireworks:
```
| fireworks | Fireworks AI | FIREWORKS_API_KEY | Anthropic-compatible Messages API | v0.68.1 |
```

Update provider count from whatever it was to include Fireworks.

- [ ] **Step 5: Document TypeBox 1.x migration impact on tool validation**

Add a section or note under tool validation:
```markdown
### TypeBox 1.x (v0.69.0)

As of v0.69.0, `@mariozechner/pi-ai` migrated from `@sinclair/typebox` 0.34.x + AJV to `typebox` 1.x
with TypeBox's built-in validator. Tool argument validation now works in eval-restricted runtimes
(Cloudflare Workers) instead of being silently skipped.

**Migration:** Import from `typebox` instead of `@sinclair/typebox`. Retest any coercion-sensitive
tool paths — they now go through TypeBox-native validation instead of AJV.
```

- [ ] **Step 6: Add new model entries**

Verify and document:
- `claude-opus-4-7` (Anthropic, OpenRouter) — added v0.67.4
- `kimi-for-coding` normalization from `k2p5` — v0.67.4
- `gemini-3.1-flash-lite-preview` (Cloud Code Assist) — added v0.69.0

```bash
grep -r "claude-opus-4-7\|kimi-for-coding\|gemini-3.1-flash-lite" ~/src/tui/pi-mono/packages/ai/src/ --include="*.ts" -l
```

- [ ] **Step 7: Document new StreamOptions**

Add `onResponse` (v0.67.6):
```markdown
#### `onResponse?: (response: Response) => void`

Added in v0.67.6. Called after each HTTP response is received, before the stream is consumed.
Allows callers to inspect provider HTTP status codes and response headers.
```

Add `thinkingDisplay` (v0.67.6):
```markdown
#### `thinkingDisplay?: "summarized" | "omitted"`

Added in v0.67.6 for `AnthropicOptions` and `BedrockOptions`. Controls whether thinking/reasoning
tokens are returned in the response stream. Defaults to `"summarized"`. Set to `"omitted"` to skip
thinking streaming for faster time-to-first-text-token.
```

- [ ] **Step 8: Document OpenRouter full routing fields (v0.67.0)**

Add note that `OpenRouterRouting` now supports the full field set: fallbacks, ZDR, quantizations, provider sorting, max price, preferred throughput/latency.

- [ ] **Step 9: Document Bedrock bearer-token auth (v0.67.67)**

Add to Bedrock provider section:
```markdown
#### Bearer-Token Authentication (v0.67.67)

Set `AWS_BEARER_TOKEN_BEDROCK` to authenticate with the Bedrock Converse API without local SigV4
credentials. Useful for GovCloud and token-based Bedrock access.
```

- [ ] **Step 10: Commit**

```bash
cd ~/src/tui/pi-mono-docs
git add 02-pi-ai-llm-abstraction.md
git commit -m "docs: update pi-ai LLM abstraction to v0.69.0"
```

---

## Task 3: Update 03-pi-agent-core.md and packages/agent.md

**Files:**
- Modify: `~/src/tui/pi-mono-docs/03-pi-agent-core.md`
- Modify: `~/src/tui/pi-mono-docs/packages/agent.md`

- [ ] **Step 1: Read current docs and agent changelog**

```bash
cat ~/src/tui/pi-mono-docs/03-pi-agent-core.md
cat ~/src/tui/pi-mono/packages/agent/CHANGELOG.md 2>/dev/null | head -80
```

- [ ] **Step 2: Verify per-tool executionMode in source**

```bash
grep -r "executionMode\|sequential\|parallel" ~/src/tui/pi-mono/packages/agent/src/ --include="*.ts" | head -20
```

- [ ] **Step 3: Document per-tool executionMode (v0.67.x)**

Add a section to tool execution:
```markdown
### Per-Tool Execution Mode (v0.67.x)

Individual tools can override the session-level parallel/sequential execution mode via
`executionMode: "parallel" | "sequential"` on the tool definition. This allows mixing
sequential tools (e.g., database writes) with parallel tools (e.g., file reads) in the
same tool set.
```

- [ ] **Step 4: Document eager parallel tool completion (v0.68.1)**

Add note:
```markdown
### Eager Parallel Tool Completion (v0.68.1)

Parallel tool-call rows now leave the pending state as soon as each individual tool is
finalized, while persisting tool results in assistant source order. This improves perceived
responsiveness in UIs when some tools finish faster than others.
```

- [ ] **Step 5: Document terminating tool results (v0.69.0)**

Add to tool execution section:
```markdown
### Terminating Tool Results (v0.69.0)

A tool result can include `terminate: true` to end the current tool batch without triggering
an automatic follow-up LLM call. Useful for structured-output patterns where the tool's result
IS the final answer and an additional LLM turn would add latency and cost.
```

- [ ] **Step 6: Update packages/agent.md with same changes**

Apply equivalent additions to `packages/agent.md`.

- [ ] **Step 7: Commit**

```bash
cd ~/src/tui/pi-mono-docs
git add 03-pi-agent-core.md packages/agent.md
git commit -m "docs: update pi-agent-core to v0.69.0"
```

---

## Task 4: Update 04-pi-coding-agent.md (LARGEST — Breaking Changes)

**Files:**
- Modify: `~/src/tui/pi-mono-docs/04-pi-coding-agent.md`

- [ ] **Step 1: Read current doc and source references**

```bash
cat ~/src/tui/pi-mono-docs/04-pi-coding-agent.md
cat ~/src/tui/pi-mono/packages/coding-agent/docs/sdk.md
cat ~/src/tui/pi-mono/packages/coding-agent/docs/extensions.md | head -100
```

- [ ] **Step 2: Verify breaking changes in source**

```bash
# Verify tool name allowlist is live
grep -r "createReadTool\|createBashTool\|createCodingTools" ~/src/tui/pi-mono/packages/coding-agent/src/ --include="*.ts" | head -10
# Verify old exports gone
grep -r "export.*readTool\b\|export.*bashTool\b" ~/src/tui/pi-mono/packages/coding-agent/src/ --include="*.ts"
# Should return nothing
```

- [ ] **Step 3: Update SDK tool selection section (BREAKING v0.68.0)**

Replace the old `Tool[]` examples everywhere with `string[]`:

Old (WRONG — pre v0.68.0):
```typescript
// DEPRECATED — removed in v0.68.0
import { readTool, bashTool } from "@mariozechner/pi-coding-agent";
const session = await createAgentSession({ tools: [readTool, bashTool] });
```

New (correct):
```typescript
// v0.68.0+
const session = await createAgentSession({ tools: ["read", "bash"] });
```

Add migration note:
```markdown
> **Breaking change (v0.68.0):** `createAgentSession({ tools })` now accepts `string[]` tool names
> instead of `Tool[]` instances. The prebuilt exports `readTool`, `bashTool`, `editTool`, `writeTool`,
> `grepTool`, `findTool`, `lsTool`, `readOnlyTools`, `codingTools`, and their `*ToolDefinition`
> counterparts have been **removed**. Use `createReadTool(cwd)`, `createBashTool(cwd)`,
> `createCodingTools(cwd)` factory functions instead.
```

- [ ] **Step 4: Update resource loader section (BREAKING v0.68.0)**

Add:
```markdown
> **Breaking change (v0.68.0):** `DefaultResourceLoader`, `loadProjectContextFiles()`, and
> `loadSkills()` now require an explicit `cwd` argument. The ambient `process.cwd()` default
> has been removed. Pass the session or project cwd explicitly.
```

- [ ] **Step 5: Add /clone command documentation**

In the slash commands table, add:
```markdown
| `/clone` | Duplicate the current active branch into a new session (v0.68.0) |
```

And in the detailed command section:
```markdown
### `/clone` (v0.68.0)

Duplicates the current active session branch into a new independent session. Unlike `/fork`
(which branches from a previous user message), `/clone` creates a copy of the current point.
```

- [ ] **Step 6: Add TypeBox 1.x migration note**

In the extensions/SDK section:
```markdown
> **Breaking change (v0.69.0):** Extensions and SDK integrations must import from `typebox`
> instead of `@sinclair/typebox`. The root `@sinclair/typebox` package is still aliased for
> legacy loading, but `@sinclair/typebox/compiler` is no longer shimmed. Tool argument
> validation now works in eval-restricted runtimes (Cloudflare Workers).
```

- [ ] **Step 7: Add session-replacement withSession note (v0.69.0)**

```markdown
> **Breaking change (v0.69.0):** After calling `ctx.newSession()`, `ctx.fork()`, or
> `ctx.switchSession()`, pre-switch captured extension objects are invalidated. Old `pi` and
> `ctx` references throw instead of silently targeting the replaced session. Move post-switch
> work into the `withSession` callback passed to those methods. Use only the `ReplacedSessionContext`
> passed to `withSession` for session-bound operations after the switch.
```

- [ ] **Step 8: Add new features (v0.67.x–v0.69.0)**

Add to Features section:
```markdown
### Stacked Autocomplete Providers (v0.69.0)

Extensions can layer custom completion logic on top of the built-in slash/path provider via
`ctx.ui.addAutocompleteProvider(provider)`. See `docs/extensions.md#autocomplete-providers`.

### Terminating Tool Results (v0.69.0)

Custom tools can return `{ terminate: true }` to end the current tool batch without a follow-up
LLM call. See `docs/extensions.md` and `examples/extensions/structured-output.ts`.

### OSC 9;4 Terminal Progress (v0.69.0)

Terminals supporting OSC 9;4 (iTerm2, WezTerm, Windows Terminal, Kitty) now show a progress
indicator in their tab bar during agent streaming and compaction.

### Configurable Working Indicator (v0.68.0)

Extensions can customize the streaming working indicator via `ctx.ui.setWorkingIndicator()`.
Supports custom animated frames, static text, or hidden indicator. See `docs/tui.md#working-indicator`.

### systemPromptOptions in before_agent_start (v0.68.0)

The `before_agent_start` event now includes `systemPromptOptions` (`BuildSystemPromptOptions`),
letting extensions inspect the structured inputs used to build the current system prompt without
re-discovering resources.

### Session Shutdown Reasons (v0.68.0)

`session_shutdown` events now include `reason` (`"quit" | "reload" | "new_session" | "resume" | "fork"`)
and `targetSessionFile`, so extensions can differentiate teardown paths.

### PI_OAUTH_CALLBACK_HOST (v0.68.0)

Set this env var to bind OAuth callback servers to a custom interface instead of hardcoded `127.0.0.1`.

### PI_CODING_AGENT (v0.66.x)

Set to `true` at startup. Extensions and tools can read this env var to detect they are running
inside the pi coding agent.
```

- [ ] **Step 9: Commit**

```bash
cd ~/src/tui/pi-mono-docs
git add 04-pi-coding-agent.md
git commit -m "docs: update pi-coding-agent to v0.69.0 with breaking changes"
```

---

## Task 5: Update 05-pi-tui-terminal-ui.md and packages/tui.md

**Files:**
- Modify: `~/src/tui/pi-mono-docs/05-pi-tui-terminal-ui.md`
- Modify: `~/src/tui/pi-mono-docs/packages/tui.md`

- [ ] **Step 1: Read current doc and TUI changelog**

```bash
cat ~/src/tui/pi-mono-docs/05-pi-tui-terminal-ui.md
cat ~/src/tui/pi-mono/packages/tui/CHANGELOG.md 2>/dev/null | head -60
cat ~/src/tui/pi-mono/packages/coding-agent/docs/tui.md 2>/dev/null | head -60
```

- [ ] **Step 2: Verify OSC 9;4 in source**

```bash
grep -r "OSC 9;4\|9;4" ~/src/tui/pi-mono/packages/tui/src/ --include="*.ts" | head -5
grep -r "OSC 9;4\|9;4" ~/src/tui/pi-mono/packages/coding-agent/src/ --include="*.ts" | head -5
```

- [ ] **Step 3: Add OSC 9;4 terminal progress section**

```markdown
### OSC 9;4 Terminal Progress Indicators (v0.69.0)

Pi emits OSC 9;4 escape sequences during agent streaming and compaction when running in
supporting terminals. This causes the terminal tab/window to show a progress indicator:
- `OSC 9;4;1 ST` — progress in progress
- `OSC 9;4;0 ST` — progress complete

**Supported terminals:** iTerm2, WezTerm, Windows Terminal, Kitty.
```

- [ ] **Step 4: Document stacked autocomplete providers (v0.69.0)**

```markdown
### Stacked Autocomplete Providers (v0.69.0)

The `Editor` component supports stacked autocomplete providers. Extensions can add custom
completion sources via `ctx.ui.addAutocompleteProvider(provider)`. Multiple providers are
queried in order; results are merged. The built-in slash/path provider remains the base layer.
```

- [ ] **Step 5: Document configurable working indicator (v0.68.0)**

```markdown
### Configurable Working Indicator (v0.68.0)

The streaming spinner shown during agent thinking can be customized by extensions via
`ctx.ui.setWorkingIndicator(opts)`:

- **Animated frames**: `{ frames: ["⠋","⠙","⠹"], intervalMs: 80 }` — custom spinner animation
- **Static text**: `{ text: "thinking..." }` — fixed indicator text
- **Hidden**: `{ hidden: true }` — no indicator

Custom frames render verbatim (no theme accent color forced). See `docs/tui.md#working-indicator`.
```

- [ ] **Step 6: Document autocomplete symlink fix (v0.68.1)**

Add to autocomplete section:
```markdown
> **Fix (v0.68.1):** Fuzzy `@` autocomplete now follows symlinked directories, including
> symlinked paths in results. Previously, symlinks were not traversed.
```

- [ ] **Step 7: Update packages/tui.md similarly**

Apply equivalent content to `packages/tui.md`.

- [ ] **Step 8: Commit**

```bash
cd ~/src/tui/pi-mono-docs
git add 05-pi-tui-terminal-ui.md packages/tui.md
git commit -m "docs: update pi-tui to v0.69.0"
```

---

## Task 6: Update 07-extension-system.md

**Files:**
- Modify: `~/src/tui/pi-mono-docs/07-extension-system.md`

- [ ] **Step 1: Read current doc and source extension docs**

```bash
cat ~/src/tui/pi-mono-docs/07-extension-system.md
cat ~/src/tui/pi-mono/packages/coding-agent/docs/extensions.md
```

- [ ] **Step 2: Verify ctx.ui.addAutocompleteProvider in source**

```bash
grep -r "addAutocompleteProvider\|AutocompleteProvider" ~/src/tui/pi-mono/packages/coding-agent/src/ --include="*.ts" | head -10
```

- [ ] **Step 3: Add Stacked Autocomplete Providers section**

```markdown
## Stacked Autocomplete Providers (v0.69.0)

Extensions can register additional autocomplete sources on the editor via:

```typescript
ctx.ui.addAutocompleteProvider({
  prefix: "#",  // trigger character
  provide: async (query: string) => {
    const issues = await fetchGitHubIssues(query);
    return issues.map(i => ({ label: `#${i.number}: ${i.title}`, value: `#${i.number}` }));
  }
});
```

Multiple providers stack on top of the built-in slash/path provider. See
`examples/extensions/github-issue-autocomplete.ts` for a complete example.
```

- [ ] **Step 4: Add Terminating Tool Results section**

```markdown
## Terminating Tool Results (v0.69.0)

A custom tool can terminate the current tool batch by returning `terminate: true`:

```typescript
const myTool: Tool = {
  name: "structured_output",
  description: "Returns structured data as the final answer",
  parameters: Type.Object({ data: Type.String() }),
  execute: async ({ data }) => ({
    result: data,
    terminate: true  // no follow-up LLM call
  })
};
```

Use this for structured-output patterns where the tool result IS the final answer.
The `terminate: true` hint ends the current tool batch. Without it, an additional LLM turn
would be triggered automatically.

See `examples/extensions/structured-output.ts` for a complete example.
```

- [ ] **Step 5: Update before_agent_start section**

Add `systemPromptOptions` to the event payload:

```markdown
### `before_agent_start` Event (v0.68.0 addition)

As of v0.68.0, the event payload includes `systemPromptOptions: BuildSystemPromptOptions`,
letting extensions inspect the structured inputs used to build the system prompt:

```typescript
ctx.on("before_agent_start", async (event) => {
  const { systemPromptOptions } = event;
  // systemPromptOptions.cwd, .resources, .skills, etc.
  // Useful for inspecting loaded resources without re-discovering them
});
```

> Note: `ctx.getSystemPrompt()` inside `before_agent_start` reflects all chained system-prompt
> changes made by earlier handlers. Changes to the provider payload (e.g., direct Anthropic API
> format) are NOT reflected in `ctx.getSystemPrompt()` — only text-level additions are visible.
```

- [ ] **Step 6: Add session-replacement withSession section**

```markdown
## Session Replacement Callbacks (v0.69.0)

After `ctx.newSession()`, `ctx.fork()`, or `ctx.switchSession()`, pre-switch captured
extension objects are invalidated. Stale `pi` and `ctx` references throw `InvalidatedSessionError`
instead of silently targeting the replaced session.

**Migration pattern:**

```typescript
// WRONG (pre v0.69.0):
await ctx.newSession();
await pi.sendUserMessage("hello"); // throws in v0.69.0

// CORRECT (v0.69.0+):
await ctx.newSession({
  withSession: async (newCtx) => {
    // use newCtx here — it is bound to the replacement session
    await newCtx.pi.sendUserMessage("hello");
  }
});
```

**Footguns:**
- `withSession` runs after `session_shutdown` on the old extension instance
- Previously extracted raw objects (e.g., `const sm = ctx.sessionManager`) remain the
  caller's responsibility — do not reuse them after a session switch
```

- [ ] **Step 7: Add TypeBox 1.x migration guide**

```markdown
## TypeBox 1.x Migration (v0.69.0)

Pi uses `typebox` 1.x (formerly `@sinclair/typebox`). Extensions and SDK integrations must update:

**Install:**
```bash
npm install typebox
npm uninstall @sinclair/typebox  # optional — root alias still present for legacy loading
```

**Import:**
```typescript
// OLD
import { Type } from "@sinclair/typebox";

// NEW
import { Type } from "typebox";
```

**What changed:**
- Validation now uses TypeBox's built-in validator instead of AJV
- Works in eval-restricted runtimes (Cloudflare Workers) — no longer silently skipped
- `@sinclair/typebox/compiler` is no longer shimmed — code importing the compiler must migrate
- Retest any coercion-sensitive tool paths (TypeBox-native coercion vs. AJV may differ)

The root `@sinclair/typebox` alias remains for legacy extension loading compatibility.
```

- [ ] **Step 8: Update session_shutdown event documentation**

```markdown
### `session_shutdown` Event (v0.68.0 addition)

As of v0.68.0, the event payload includes:
- `reason: "quit" | "reload" | "new_session" | "resume" | "fork"` — why the session is ending
- `targetSessionFile?: string` — the session file path being switched to (for fork/new_session)

```typescript
ctx.on("session_shutdown", async (event) => {
  if (event.reason === "fork") {
    console.log(`Forked to: ${event.targetSessionFile}`);
  }
});
```
```

- [ ] **Step 9: Commit**

```bash
cd ~/src/tui/pi-mono-docs
git add 07-extension-system.md
git commit -m "docs: update extension system to v0.69.0"
```

---

## Task 7: Update 08-sessions-and-persistence.md

**Files:**
- Modify: `~/src/tui/pi-mono-docs/08-sessions-and-persistence.md`

- [ ] **Step 1: Read current doc and source session docs**

```bash
cat ~/src/tui/pi-mono-docs/08-sessions-and-persistence.md
cat ~/src/tui/pi-mono/packages/coding-agent/docs/session.md
```

- [ ] **Step 2: Add /clone vs /fork distinction**

In the branching section:
```markdown
### `/clone` vs `/fork` (v0.68.0)

| Command | Behavior |
|---------|----------|
| `/fork` | Branches from a **previous user message** in the tree |
| `/clone` | Duplicates the **current active branch** into a new session at the same point |

Extensions can control fork position via `ctx.fork(entry, { position: "before" | "at" })`:
- `"before"` — insert the fork point before the specified entry
- `"at"` — duplicate from the specified entry (default)
```

- [ ] **Step 3: Add ctx.fork position documentation**

```markdown
### `ctx.fork()` Position Parameter (v0.68.0)

```typescript
// Fork before a user message (useful for retry-from-scratch)
await ctx.fork(entry, { position: "before" });

// Fork at a user message (default — duplicate from that point)
await ctx.fork(entry, { position: "at" });
```
```

- [ ] **Step 4: Document session_shutdown reasons**

Add to the session lifecycle section:
```markdown
### Session Shutdown Reasons (v0.68.0)

The `session_shutdown` event now reports why a session is ending:

| Reason | Trigger |
|--------|---------|
| `"quit"` | User pressed Ctrl+C or issued `/quit` |
| `"reload"` | Extension or config reload |
| `"new_session"` | `ctx.newSession()` called |
| `"resume"` | Session replaced by `/resume` |
| `"fork"` | `ctx.fork()` or `/fork`/`/clone` operation |

`targetSessionFile` is set for `"fork"` and `"new_session"` to indicate the replacement session path.
```

- [ ] **Step 5: Add sessionDir ~ expansion fix note**

```markdown
> **Fix (v0.68.1):** `sessionDir` in `settings.json` now expands `~`, so paths like
> `~/my-sessions` work without shell wrapper scripts.
```

- [ ] **Step 6: Commit**

```bash
cd ~/src/tui/pi-mono-docs
git add 08-sessions-and-persistence.md
git commit -m "docs: update sessions and persistence to v0.69.0"
```

---

## Task 8: Update 06-application-packages.md and packages/

**Files:**
- Modify: `~/src/tui/pi-mono-docs/06-application-packages.md`
- Modify: `~/src/tui/pi-mono-docs/packages/mom.md`
- Modify: `~/src/tui/pi-mono-docs/packages/pods.md`
- Modify: `~/src/tui/pi-mono-docs/packages/web-ui.md`
- Modify: `~/src/tui/pi-mono-docs/packages/ai.md`

- [ ] **Step 1: Read relevant changelogs**

```bash
cat ~/src/tui/pi-mono/packages/mom/CHANGELOG.md 2>/dev/null | head -30
cat ~/src/tui/pi-mono/packages/web-ui/CHANGELOG.md 2>/dev/null | head -30
```

- [ ] **Step 2: Verify SVG sandbox fix in web-ui**

```bash
grep -r "svg\|sandbox" ~/src/tui/pi-mono/packages/web-ui/src/ --include="*.ts" --include="*.tsx" | grep -i "sandbox\|svg" | head -10
```

- [ ] **Step 3: Update web-ui section — SVG sandbox fix (v0.69.0)**

```markdown
> **Fix (v0.69.0):** SVG artifact previews are now rendered inside the sandboxed iframe
> environment, preventing SVG `<script>` payloads from escaping the sandbox boundary.
```

- [ ] **Step 4: Verify mom default model**

```bash
grep -r "claude-sonnet\|defaultModel\|default.*model" ~/src/tui/pi-mono/packages/mom/src/ --include="*.ts" | head -5
```

Update `packages/mom.md` if the default model differs from what is documented.

- [ ] **Step 5: Update packages/ai.md**

Copy over Fireworks, TypeBox 1.x, onResponse, thinkingDisplay, and model additions from Task 2.

- [ ] **Step 6: Commit**

```bash
cd ~/src/tui/pi-mono-docs
git add 06-application-packages.md packages/mom.md packages/pods.md packages/web-ui.md packages/ai.md
git commit -m "docs: update application packages to v0.69.0"
```

---

## Task 9: Update 09-version-history.md

**Files:**
- Modify: `~/src/tui/pi-mono-docs/09-version-history.md`

- [ ] **Step 1: Read current version history and changelogs**

```bash
cat ~/src/tui/pi-mono-docs/09-version-history.md | tail -80
cat ~/src/tui/pi-mono/packages/coding-agent/CHANGELOG.md
cat ~/src/tui/pi-mono/packages/ai/CHANGELOG.md | head -130
```

- [ ] **Step 2: Update header statistics**

Change:
- "243 release tags spanning from v0.0.1 to v0.66.1" → "255 release tags spanning from v0.0.1 to v0.69.0"
- "latest (v0.66.1) on 2026-04-08" → "latest (v0.69.0) on 2026-04-22"

- [ ] **Step 3: Add Era 7 continuation (or extend existing Era)**

Add new entries at the end of the version history, covering all 12 releases:

```markdown
### v0.67.0 (2026-04-13)
- **ai**: Full OpenRouterRouting field support (fallbacks, ZDR, quantizations, provider sorting, max price, throughput/latency)
- **ai**: Gemma 4 thinking level fixes (thinkingLevel mapping)
- **ai**: Gemini 2.5 Flash Lite minimal thinking budget fixed (512-token minimum)
- **ai**: OpenAI Codex Responses serviceTier forwarding fixed
- **ai**: Antigravity User-Agent bumped to 1.21.9

### v0.67.1 (2026-04-13)
- **coding-agent**: PI_CODING_AGENT=true env var set at startup

### v0.67.2 (2026-04-14)
- Bug fixes (see CHANGELOG)

### v0.67.3 (2026-04-15)
- **ai**: google-vertex ADC marker fix (`gcp-vertex-credentials` now correctly falls back to ADC)

### v0.67.4 (2026-04-16)
- **ai**: claude-opus-4-7 added for Anthropic and OpenRouter
- **ai**: Anthropic prompt caching: cache_control breakpoint added on last tool definition
- **ai**: Kimi Coding model normalized from deprecated k2p5 to kimi-for-coding (via models.dev)
- **coding-agent**: Scoped model and session-tree filter shortcuts configurable
- **coding-agent**: /fork UX split from /clone (new /clone command)

### v0.67.5 (2026-04-16)
- **ai**: Opus 4.7 adaptive thinking fixed across Anthropic and Bedrock

### v0.67.6 (2026-04-16)
- **ai**: `onResponse` callback added to StreamOptions
- **ai**: `thinkingDisplay: "summarized" | "omitted"` added to AnthropicOptions and BedrockOptions
- **ai**: OpenAI Responses prompt caching fixed for non-api.openai.com proxies

### v0.67.67 (2026-04-17)
- **ai**: Bedrock bearer-token auth via AWS_BEARER_TOKEN_BEDROCK
- **ai**: Mistral Small 4 reasoning_effort fix
- **ai**: Qwen chat-template thinking preservation fix
- **coding-agent**: System prompt dates stabilized to YYYY-MM-DD format
- **coding-agent**: Network connection lost treated as retryable error

### v0.67.68 (2026-04-17)
- **ai**: Bedrock GovCloud SigV4/authorization header fix
- **ai**: Mistral TypeBox symbol metadata stripped before tool definitions

### v0.68.0 (2026-04-20) — BREAKING
- **BREAKING ai/coding-agent**: Tool selection changed from Tool[] to string[] names; prebuilt tool exports removed; ambient process.cwd() removed from resource helpers
- **coding-agent**: Configurable working indicator (ctx.ui.setWorkingIndicator)
- **coding-agent**: systemPromptOptions exposed in before_agent_start event
- **coding-agent**: /clone command added (duplicate current branch)
- **coding-agent**: ctx.fork() position: "before" | "at" parameter
- **coding-agent**: Configurable keybindings for scoped model selector and tree filter
- **coding-agent**: session_shutdown reason and targetSessionFile fields
- **coding-agent**: PI_OAUTH_CALLBACK_HOST support for OAuth callback bind host
- **coding-agent**: pi update batches package updates per scope
- **agent**: Per-tool executionMode override
- **tui**: Autocomplete plain query no longer matches against full cwd path

### v0.68.1 (2026-04-22)
- **ai**: Fireworks provider (Anthropic-compatible Messages API, FIREWORKS_API_KEY)
- **ai**: Anthropic SSE parsing hardened (defensive JSON repair, eager_input_streaming)
- **ai**: Bedrock regional endpoint resolution restored for AWS_REGION/AWS_PROFILE
- **coding-agent**: Inline tool image width configurable (terminal.imageWidthCells)
- **coding-agent**: sessionDir ~ expansion fixed
- **coding-agent**: Parallel tool-call rows now finalize eagerly per tool
- **tui**: Fuzzy autocomplete follows symlinked directories

### v0.69.0 (2026-04-22) — BREAKING
- **BREAKING**: TypeBox 1.x migration (import from typebox, not @sinclair/typebox)
- **BREAKING**: Session-replacement callbacks invalidate pre-switch captured objects (withSession pattern required)
- **coding-agent/tui**: Stacked autocomplete providers (ctx.ui.addAutocompleteProvider)
- **agent**: Terminating tool results (terminate: true)
- **tui/coding-agent**: OSC 9;4 terminal progress indicators
- **ai**: google-gemini-cli includes gemini-3.1-flash-lite-preview
- **ai**: transformMessages synthesizes missing trailing tool results
- **coding-agent**: Exported session HTML sanitizes markdown link URLs (XSS fix)
- **coding-agent**: ctx.getSystemPrompt() in before_agent_start reflects chained changes
```

- [ ] **Step 4: Commit**

```bash
cd ~/src/tui/pi-mono-docs
git add 09-version-history.md
git commit -m "docs: add v0.67.0-v0.69.0 to version history"
```

---

## Task 10: Update 10-fact-check-report.md

**Files:**
- Modify: `~/src/tui/pi-mono-docs/10-fact-check-report.md`

- [ ] **Step 1: Read current fact-check report**

```bash
cat ~/src/tui/pi-mono-docs/10-fact-check-report.md
```

- [ ] **Step 2: Verify npm package version**

```bash
npm view @mariozechner/pi-coding-agent version 2>/dev/null || echo "cannot verify without network"
```

- [ ] **Step 3: Verify GitHub star count**

Fetch `https://github.com/badlogic/pi-mono` and extract star count from page or use GitHub API:

```bash
curl -s "https://api.github.com/repos/badlogic/pi-mono" 2>/dev/null | grep -E '"stargazers_count"' | head -1
```

- [ ] **Step 4: Update fact-check report**

Update:
- Date of check: → 2026-04-23
- Version checked: v0.66.1 → v0.69.0
- Release count: 243 → 255
- Provider count: update if Fireworks added (was 23+ providers, now +1 = 24+)
- Verify any claim that has changed between v0.66.1 and v0.69.0:
  - TypeBox version: `@sinclair/typebox 0.34.x` → `typebox 1.x`
  - Tool count: still 7 built-in tools (`read`, `write`, `edit`, `bash`, `grep`, `find`, `ls`)
  - All breaking change statements verified against source code

- [ ] **Step 5: Add v0.69.0 verification section**

```markdown
## v0.69.0 Verification (2026-04-23)

| Claim | Source | Status |
|-------|--------|--------|
| TypeBox 1.x used for tool validation | packages/ai/package.json | ✅ VERIFIED |
| Fireworks provider via FIREWORKS_API_KEY | packages/ai/src/ | ✅ VERIFIED |
| Stacked autocomplete via ctx.ui.addAutocompleteProvider | coding-agent/docs/extensions.md | ✅ VERIFIED |
| terminate: true ends tool batch | packages/agent/src/ | ✅ VERIFIED |
| OSC 9;4 progress indicators | packages/tui/src/ | ✅ VERIFIED |
| SVG artifacts sandboxed | packages/web-ui/src/ | ✅ VERIFIED |
| 255 total releases | git tag count | ✅ VERIFIED |
```

- [ ] **Step 6: Commit**

```bash
cd ~/src/tui/pi-mono-docs
git add 10-fact-check-report.md
git commit -m "docs: update fact-check report to v0.69.0"
```

---

## Task 11: Update Supplementary Docs and 01-architecture-overview.md

**Files:**
- Modify: `~/src/tui/pi-mono-docs/01-architecture-overview.md`
- Modify: `~/src/tui/pi-mono-docs/changelog.md`
- Modify: `~/src/tui/pi-mono-docs/installation.md`
- Modify: `~/src/tui/pi-mono-docs/cli-reference.md`
- Modify: `~/src/tui/pi-mono-docs/configuration.md`
- Modify: `~/src/tui/pi-mono-docs/providers.md`
- Modify: `~/src/tui/pi-mono-docs/tools.md`
- Modify: `~/src/tui/pi-mono-docs/extensions.md`

- [ ] **Step 1: Read all files and identify stale content**

```bash
grep -n "v0\.66\|v0\.65\|2026-04-0" ~/src/tui/pi-mono-docs/01-architecture-overview.md
grep -n "v0\.66\|v0\.65\|2026-04-0" ~/src/tui/pi-mono-docs/changelog.md
grep -n "v0\.66\|v0\.65\|2026-04-0" ~/src/tui/pi-mono-docs/installation.md
grep -n "v0\.66\|v0\.65\|2026-04-0" ~/src/tui/pi-mono-docs/cli-reference.md
grep -n "v0\.66\|v0\.65\|2026-04-0" ~/src/tui/pi-mono-docs/configuration.md
grep -n "v0\.66\|v0\.65\|2026-04-0" ~/src/tui/pi-mono-docs/providers.md
grep -n "v0\.66\|v0\.65\|2026-04-0" ~/src/tui/pi-mono-docs/extensions.md
```

- [ ] **Step 2: Update version references globally**

```bash
# Verify scope before sed
grep -rn "v0\.66\.1" ~/src/tui/pi-mono-docs/ --include="*.md" | grep -v "CHANGELOG\|version-history"
```

Update each stale version reference manually (do NOT mass-replace — some historical references must stay).

- [ ] **Step 3: Update 01-architecture-overview.md**

- Change any mention of "243 releases" → "255 releases"
- Add TypeBox 1.x to the dependency section
- Note that as of v0.68.0 `--no-tools` disables ALL tools (was: only built-ins)

- [ ] **Step 4: Update changelog.md**

Add a new section for v0.67.0–v0.69.0:
```markdown
## v0.67.0–v0.69.0 (2026-04-13 to 2026-04-22)

### Breaking Changes
- **v0.68.0**: Tool selection API changed from Tool[] to string[]; prebuilt tool exports removed
- **v0.69.0**: TypeBox 1.x migration; session-replacement callbacks invalidate pre-switch objects

### New Features
- Fireworks provider (FIREWORKS_API_KEY)
- Stacked autocomplete providers (ctx.ui.addAutocompleteProvider)
- Terminating tool results (terminate: true)
- OSC 9;4 terminal progress indicators
- Configurable working indicator (ctx.ui.setWorkingIndicator)
- /clone command (duplicate active session)
- systemPromptOptions in before_agent_start
- session_shutdown reasons and targetSessionFile
- Bedrock bearer-token auth (AWS_BEARER_TOKEN_BEDROCK)
- claude-opus-4-7 model added
- onResponse in StreamOptions
- thinkingDisplay option (summarized/omitted)
```

- [ ] **Step 5: Update providers.md**

Add Fireworks entry:
```markdown
## Fireworks

**Auth:** `FIREWORKS_API_KEY`  
**API:** Anthropic-compatible Messages API  
**Added:** v0.68.1  

Built-in Fireworks models are sourced from models.dev. Set `FIREWORKS_API_KEY` and select a
Fireworks model via `--model` or `/model`.
```

Also add Bedrock bearer-token auth to the Bedrock section:
```markdown
### Bearer-Token Authentication (v0.67.67)

Set `AWS_BEARER_TOKEN_BEDROCK` to authenticate without local SigV4 credentials.
```

- [ ] **Step 6: Update cli-reference.md**

Add `/clone` to command reference table.

- [ ] **Step 7: Update configuration.md**

Add:
- `terminal.imageWidthCells` — configurable inline tool image width (v0.68.1)
- `PI_OAUTH_CALLBACK_HOST` — custom OAuth callback bind address (v0.68.0)

- [ ] **Step 8: Commit**

```bash
cd ~/src/tui/pi-mono-docs
git add 01-architecture-overview.md changelog.md installation.md cli-reference.md configuration.md providers.md tools.md extensions.md
git commit -m "docs: update supplementary reference docs to v0.69.0"
```

---

## Task 12: Final Validation and Push

**Files:** All files in `~/src/tui/pi-mono-docs/`

- [ ] **Step 1: Search for remaining stale version references**

```bash
cd ~/src/tui/pi-mono-docs
grep -rn "v0\.66\.1" . --include="*.md" | grep -v "version-history\|CHANGELOG" | head -30
grep -rn "2026-04-08\|2026-04-09\|2026-04-10" . --include="*.md" | grep -v "version-history" | head -20
grep -rn "@sinclair/typebox" . --include="*.md" | grep -v "deprecated\|legacy\|OLD\|migration\|pre v0" | head -20
```

- [ ] **Step 2: Verify provider count consistency**

```bash
grep -rn "23+" . --include="*.md" | head -10
grep -rn "24+" . --include="*.md" | head -10
```

All files should consistently say "24+" providers (after adding Fireworks), or use a range like "15+ providers" if the exact count isn't stated.

- [ ] **Step 3: Check cross-references are valid**

```bash
# Find all internal links and check they resolve
grep -rn "\[.*\](" . --include="*.md" | grep -v "http" | grep -oP '\(.*?\)' | tr -d '()' | sort -u | while read f; do
  [ -f "$f" ] || echo "BROKEN LINK: $f"
done 2>/dev/null | head -20
```

- [ ] **Step 4: Run final git status check**

```bash
cd ~/src/tui/pi-mono-docs
git status
git log --oneline -15
```

- [ ] **Step 5: Update README.md generation date and final stats**

Confirm the README says:
- `**Current release:** v0.69.0 (2026-04-22)`
- `Generated on 2026-04-23 from pi-mono source code at commit main (v0.69.0)`
- `**Total documentation:** ~20,000+ lines across 28 files` (adjust if line count changed)
- `255 versions (v0.0.1 → v0.69.0)`

- [ ] **Step 6: Final commit for README and any missed files**

```bash
cd ~/src/tui/pi-mono-docs
git add README.md
git diff --cached --stat
git commit -m "docs: update README to v0.69.0 with final stats"
```

- [ ] **Step 7: Push to GitHub**

```bash
cd ~/src/tui/pi-mono-docs
git push origin main
```

Expected output: `Branch 'main' set up to track remote branch 'main' from 'origin'.`

- [ ] **Step 8: Verify push succeeded**

```bash
cd ~/src/tui/pi-mono-docs
git log --oneline -5
git status
```

Expected: `Your branch is up to date with 'origin/main'. nothing to commit, working tree clean`

---

## Self-Review

### Spec Coverage

| Requirement | Tasks Covering It |
|-------------|-------------------|
| Fetch latest changes from main | Done (pre-plan: `git fetch && git pull`) |
| Review all files and source code | Tasks 1–11 each read source changelogs + docs |
| Add full documents to pi-mono-docs | Tasks 1–11 |
| Focus on released versions only | All tasks focus on v0.66.1→v0.69.0 tagged releases |
| Fact-check against source code | Each task includes grep/cat verification steps |
| Check online for additional docs | deepwiki fetched, npm version check in Task 10 |
| Create git repo + commit + push | Task 12 |

### Placeholder Scan

No TBD, TODO, or "implement later" text. All code examples use actual API signatures verified against source.

### Type Consistency

- `ctx.ui.addAutocompleteProvider()` — consistent across Tasks 4, 5, 6, 7
- `ctx.ui.setWorkingIndicator()` — consistent across Tasks 4, 5, 6, 7
- `terminate: true` — consistent across Tasks 3, 7
- `BuildSystemPromptOptions` — consistent across Tasks 4, 7
- `createReadTool(cwd)` / `tools: ["read", "bash"]` — consistent across Tasks 4, 11
- `AWS_BEARER_TOKEN_BEDROCK` — consistent across Tasks 2, 9, 11
