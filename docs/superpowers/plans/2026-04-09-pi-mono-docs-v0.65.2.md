# Pi-Mono Documentation Update: v0.60.0 → v0.65.2

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Update all documentation in `~/src/tui/pi-mono-docs` from v0.60.0 to v0.65.2, incorporating all breaking changes, new features, and bug-fix notes across 9 release tags (v0.61.0–v0.65.2).

**Architecture:** Six parallel agent groups each own a doc slice — core API, application layer, infrastructure, version/reference, supplementary reference, and package docs. Each agent reads source from `~/src/tui/pi-mono` plus deepwiki, then writes updated markdown to `~/src/tui/pi-mono-docs`. A final agent commits and pushes to `github.com/mudrii/pi-mono-docs`.

**Tech Stack:** TypeScript/Node.js monorepo (npm workspaces), JSONL sessions, TypeBox schemas, Biome linter. Docs in Markdown. Git repo: `https://github.com/mudrii/pi-mono-docs.git`.

---

## Release summary: v0.60.0 → v0.65.2

Key breaking changes (must be documented accurately):

### v0.63.0
- `ModelRegistry.getApiKey(model)` removed → `getApiKeyAndHeaders(model)` (returns `{apiKey, headers}`)
- Deprecated `minimax` / `minimax-cn` model IDs removed; use `MiniMax-M2.7` or `MiniMax-M2.7-highspeed`

### v0.64.0
- `ModelRegistry` public constructor removed → use `ModelRegistry.create(authStorage, modelsJsonPath?)` or `ModelRegistry.inMemory(authStorage)`

### v0.65.0
- `AgentState` reshape: `streamMessage` → `streamingMessage`, `error` → `errorMessage`; `isStreaming`, `streamingMessage`, `pendingToolCalls`, `errorMessage` are now readonly; `pendingToolCalls` typed as `ReadonlySet<string>`; `tools`/`messages` are now accessor properties (assignment copies the array)
- All `Agent` mutator methods removed: `setSystemPrompt`, `setModel`, `setThinkingLevel`, `setTools`, `replaceMessages`, `appendMessage`, `clearMessages`, `setToolExecution`, `setBeforeToolCall`, `setAfterToolCall`, `setTransport` → replaced by direct property assignment (`agent.state.systemPrompt = ...`)
- Queue mode methods removed: `setSteeringMode(mode)` → `agent.steeringMode`, `getSteeringMode()` → `agent.steeringMode`, `setFollowUpMode(mode)` → `agent.followUpMode`
- `Agent.subscribe()` listeners now async and receive `(event, signal)`. `agent_end` is final emitted event. `waitForIdle()`, `prompt()`, `continue()` settle after awaited `agent_end` listeners
- Extension events `session_switch` and `session_fork` removed → use `session_start` with `event.reason` (`"startup" | "reload" | "new" | "resume" | "fork"`) and `event.previousSessionFile`
- `session_directory` removed from extension and settings APIs
- Unknown single-dash CLI flags now error instead of being silently ignored
- `AgentSessionRuntime` / `createAgentSessionRuntime()` added for SDK session replacement

New features v0.61–v0.65:
- `ctx.signal` in ExtensionContext (v0.63.2)
- `edit` tool unified to `edits[]` schema only (v0.63.2)
- `ToolDefinition.prepareArguments` hook (v0.64.0)
- `ctx.ui.setHiddenThinkingLabel()` (v0.64.0)
- Faux provider helpers for deterministic tests: `registerFauxProvider()`, `fauxAssistantMessage()`, etc. (v0.64.0)
- `defineTool()` standalone tool helper with TypeScript type inference (v0.65.0)
- Label timestamps in `/tree` with `Shift+T` toggle (v0.65.0)
- Unified diagnostics model: `info`/`warning`/`error` (v0.65.0)
- `PI_TUI_WRITE_LOG` directory support for multi-session debug logging (v0.63.0 tui)
- Bedrock `requestMetadata` for AWS cost tagging (v0.62.0)
- Z.ai tool streaming support (v0.65.0 ai)
- Render scheduling throttle (16ms budget, v0.65.2 tui)
- Anthropic subscription auth warning (v0.65.2)
- `gpt-5.4-mini` for openai-codex provider (v0.61.0)
- Anthropic HTTP 413 context overflow detection (v0.65.0)
- Ollama `prompt too long` overflow detection (v0.63.1 ai)
- `gemini-3.1-pro-preview-customtools` for google-vertex (v0.63.1)

---

## File Map

### Files to modify in `~/src/tui/pi-mono-docs/`

| File | Change type | Responsible task |
|------|------------|-----------------|
| `README.md` | Version bump (v0.60.0 → v0.65.2), date update, table update | Task 4 |
| `01-architecture-overview.md` | Version bump, note lockstep tag jump (v0.60 → v0.61, no v0.60.x tags exist between) | Task 4 |
| `02-pi-ai-llm-abstraction.md` | ModelRegistry API changes, new providers, BedrockOptions, faux provider, model catalog updates | Task 1 |
| `03-pi-agent-core.md` | AgentState reshape, mutator method removal, subscribe signature, AgentTool.prepareArguments, Agent.signal | Task 1 |
| `04-pi-coding-agent.md` | AgentSessionRuntime, defineTool, unified diagnostics, ModelRegistry constructor, session_start changes | Task 2 |
| `07-extension-system.md` | ctx.signal, session_start reason, session_switch/fork removal, prepareArguments, defineTool, setHiddenThinkingLabel | Task 2 |
| `05-pi-tui-terminal-ui.md` | PI_TUI_WRITE_LOG, render scheduling, Kitty keyboard fixes, autocomplete async fixes | Task 3 |
| `08-sessions-and-persistence.md` | AgentSessionRuntime, session cwd handling, unified diagnostics, session_directory removal | Task 3 |
| `09-version-history.md` | Add v0.61.0–v0.65.2 era section with all releases | Task 4 |
| `10-fact-check-report.md` | Re-verify npm stats, GitHub stars, domains; update for v0.65.2 | Task 5 |
| `changelog.md` | Add v0.61.0–v0.65.2 entries | Task 5 |
| `extensions.md` | session_start reason, ctx.signal, prepareArguments, defineTool, setHiddenThinkingLabel | Task 5 |
| `sessions.md` | AgentSessionRuntime, session cwd handling | Task 5 |
| `cli-reference.md` | Unknown single-dash flag errors, /exit removal, /tree Shift+T, --mode json piped stdin fix | Task 5 |
| `configuration.md` | session_directory removal, unified diagnostics | Task 5 |
| `providers.md` | Model catalog updates, gpt-5.4-mini, gemini-3.1-pro-preview-customtools, MiniMax cleanup | Task 5 |
| `tools.md` | edit tool edits[] only schema, prepareArguments, bash output truncation fix | Task 5 |
| `packages/ai.md` | ModelRegistry API, BedrockOptions requestMetadata, faux provider | Task 6 |
| `packages/agent.md` | AgentState reshape, mutator removal, Agent.signal, subscribe signature | Task 6 |
| `packages/tui.md` | PI_TUI_WRITE_LOG, render scheduling, keyboard fixes | Task 6 |
| `packages/mom.md` | Version bump only (likely no functional changes) | Task 6 |
| `packages/web-ui.md` | Version bump (check for any changes) | Task 6 |
| `packages/pods.md` | Version bump (check for any changes) | Task 6 |

---

## Task 1: Update Core API Docs (pi-ai + pi-agent-core)

**Files:**
- Modify: `~/src/tui/pi-mono-docs/02-pi-ai-llm-abstraction.md`
- Modify: `~/src/tui/pi-mono-docs/03-pi-agent-core.md`

**Source to read:**
- `~/src/tui/pi-mono/packages/ai/CHANGELOG.md` (full)
- `~/src/tui/pi-mono/packages/ai/src/index.ts`
- `~/src/tui/pi-mono/packages/agent/CHANGELOG.md` (full)
- `~/src/tui/pi-mono/packages/agent/src/types.ts`
- `~/src/tui/pi-mono/packages/agent/src/agent.ts`
- deepwiki: `https://deepwiki.com/badlogic/pi-mono/2-pi-ai-llm-abstraction`
- deepwiki: `https://deepwiki.com/badlogic/pi-mono/3-pi-agent-core`

- [ ] **Step 1: Read existing 02-pi-ai-llm-abstraction.md**

```bash
cat ~/src/tui/pi-mono-docs/02-pi-ai-llm-abstraction.md
```

- [ ] **Step 2: Read pi-ai source changes**

```bash
cat ~/src/tui/pi-mono/packages/ai/CHANGELOG.md
cat ~/src/tui/pi-mono/packages/ai/src/index.ts | head -100
grep -n "getApiKeyAndHeaders\|ModelRegistry\|FauxProvider\|registerFauxProvider\|BedrockOptions\|requestMetadata" ~/src/tui/pi-mono/packages/ai/src/index.ts
```

- [ ] **Step 3: Read pi-agent-core source changes**

```bash
cat ~/src/tui/pi-mono/packages/agent/CHANGELOG.md
cat ~/src/tui/pi-mono/packages/agent/src/types.ts
grep -n "streamingMessage\|errorMessage\|ReadonlySet\|AgentTool\|prepareArguments\|signal" ~/src/tui/pi-mono/packages/agent/src/agent.ts | head -60
```

- [ ] **Step 4: Fetch deepwiki pages**

Fetch `https://deepwiki.com/badlogic/pi-mono/2-pi-ai-llm-abstraction` and `https://deepwiki.com/badlogic/pi-mono/3-pi-agent-core`

- [ ] **Step 5: Update 02-pi-ai-llm-abstraction.md**

Update the version header from v0.60.0 to v0.65.2. Apply all changes:

**ModelRegistry section** — add breaking change note for v0.63.0 and v0.64.0:

```markdown
### ModelRegistry (v0.63.0+ breaking change)

`ModelRegistry.getApiKey(model)` was removed in v0.63.0. Use `getApiKeyAndHeaders(model)` instead:

```typescript
// Before (removed in v0.63.0)
const apiKey = await ctx.modelRegistry.getApiKey(model);

// After
const { apiKey, headers } = await ctx.modelRegistry.getApiKeyAndHeaders(model);
```

In v0.64.0, `ModelRegistry` no longer has a public constructor. Use factory methods:

```typescript
// For file-backed registries (most common)
const registry = ModelRegistry.create(authStorage);
const registry = ModelRegistry.create(authStorage, customModelsJsonPath);

// For in-memory/test registries (no file I/O)
const registry = ModelRegistry.inMemory(authStorage);
```
```

**Faux provider section** — add v0.64.0 additions:

```markdown
### Faux Provider (Testing, v0.64.0+)

For deterministic tests and scripted demos, register a faux provider:

```typescript
import {
  registerFauxProvider,
  fauxAssistantMessage,
  fauxText,
  fauxThinking,
  fauxToolCall,
} from "@mariozechner/pi-ai";

registerFauxProvider();
// Use provider: "faux" in model config
```
```

**BedrockOptions section** — add v0.62.0 requestMetadata:

```markdown
### BedrockOptions.requestMetadata (v0.62.0+)

Pass key-value pairs for AWS cost allocation tagging:

```typescript
const options: BedrockOptions = {
  requestMetadata: {
    project: "my-app",
    team: "backend",
  },
};
```

These forward to Bedrock Converse API `requestMetadata` and appear in AWS Cost Explorer split cost allocation data.
```

**Provider updates section:**
- v0.63.1: `gemini-3.1-pro-preview-customtools` available for `google-vertex`
- v0.65.0: Z.ai tool streaming support added
- v0.61.0: `gpt-5.4-mini` for `openai-codex`
- v0.61.1: MiniMax model metadata normalized
- Removed: `minimax` and `minimax-cn` direct IDs in v0.63.0

- [ ] **Step 6: Update 03-pi-agent-core.md**

Update version header to v0.65.2. Apply v0.65.0 breaking changes:

**AgentState section** (full replacement of the interface documentation):

```markdown
## AgentState (v0.65.0+)

`AgentState` was reshaped in v0.65.0:

```typescript
interface AgentState {
  // Mutable (readable and writable)
  systemPrompt: string;
  model: Model<any>;
  thinkingLevel: ThinkingLevel;

  // Accessor properties — assignment copies the provided array
  tools: AgentTool<any>[];
  messages: AgentMessage[];

  // Readonly at runtime (set by the agent loop)
  readonly isStreaming: boolean;
  readonly streamingMessage: AgentMessage | null;  // was streamMessage in ≤0.64
  readonly pendingToolCalls: ReadonlySet<string>;
  readonly errorMessage: string | undefined;        // was error in ≤0.64
}
```

**Migration note:** If your code reads `agent.state.streamMessage`, change to `agent.state.streamingMessage`. If it reads `agent.state.error`, change to `agent.state.errorMessage`.

**AgentOptions.initialState** no longer accepts `isStreaming`, `streamingMessage`, `pendingToolCalls`, or `errorMessage`. Remove them from your `initialState` objects.
```

**Agent methods section** (full replacement):

```markdown
## Agent Property API (v0.65.0+)

All `setXxx()` mutator methods were removed in v0.65.0. Use direct property assignment:

| Before (removed) | After |
|------------------|-------|
| `agent.setSystemPrompt(v)` | `agent.state.systemPrompt = v` |
| `agent.setModel(m)` | `agent.state.model = m` |
| `agent.setThinkingLevel(l)` | `agent.state.thinkingLevel = l` |
| `agent.setTools(t)` | `agent.state.tools = t` |
| `agent.replaceMessages(msgs)` | `agent.state.messages = msgs` |
| `agent.appendMessage(msg)` | `agent.state.messages.push(msg)` |
| `agent.clearMessages()` | `agent.state.messages = []` |
| `agent.setToolExecution(m)` | `agent.toolExecution = m` |
| `agent.setBeforeToolCall(fn)` | `agent.beforeToolCall = fn` |
| `agent.setAfterToolCall(fn)` | `agent.afterToolCall = fn` |
| `agent.setTransport(t)` | `agent.transport = t` |
| `agent.setSteeringMode(m)` | `agent.steeringMode = m` |
| `agent.getSteeringMode()` | `agent.steeringMode` |
| `agent.setFollowUpMode(m)` | `agent.followUpMode = m` |
| `agent.getFollowUpMode()` | `agent.followUpMode` |
```

**subscribe() section** (update for async/signal):

```markdown
## agent.subscribe() (v0.65.0+)

Listeners are now awaited and receive the active `AbortSignal`:

```typescript
// Before (≤0.64)
agent.subscribe((event) => { ... });

// After (v0.65.0+)
agent.subscribe(async (event, signal) => { ... });
```

`agent_end` is now the final emitted event. `waitForIdle()`, `prompt()`, and `continue()` settle only after awaited `agent_end` listeners complete. `agent.state.isStreaming` remains `true` until that settlement completes.
```

**Agent.signal section** (add v0.63.2 feature):

```markdown
## Agent.signal (v0.63.2+)

`agent.signal` exposes the active `AbortSignal` for the current turn. Use it to forward cancellation into nested async work:

```typescript
const response = await fetch(url, { signal: agent.signal });
```

Returns `undefined` when no agent turn is active.
```

**AgentTool.prepareArguments section** (add v0.64.0 feature):

```markdown
## AgentTool.prepareArguments (v0.64.0+)

Attach a `prepareArguments` hook to normalize raw model arguments before schema validation. Useful for backwards-compatibility with sessions saved under old schemas:

```typescript
const myTool: AgentTool<MyParams> = {
  name: "my-tool",
  description: "...",
  parameters: MyParamsSchema,
  prepareArguments: (rawArgs) => {
    // Normalize/migrate before validation
    return { ...rawArgs, normalizedField: rawArgs.oldField ?? rawArgs.normalizedField };
  },
  execute: async (params) => { ... },
};
```
```

- [ ] **Step 7: Verify fact-checks**

```bash
# Check actual field names in source
grep -n "streamingMessage\|streamMessage\|errorMessage\|ReadonlySet" ~/src/tui/pi-mono/packages/agent/src/types.ts | head -30
grep -n "subscribe\|agent_end\|waitForIdle" ~/src/tui/pi-mono/packages/agent/src/agent.ts | head -30
grep -n "getApiKeyAndHeaders\|ModelRegistry" ~/src/tui/pi-mono/packages/ai/src/index.ts | head -20
```

- [ ] **Step 8: Commit Task 1**

```bash
cd ~/src/tui/pi-mono-docs
git add 02-pi-ai-llm-abstraction.md 03-pi-agent-core.md
git commit -m "docs: update pi-ai and pi-agent-core for v0.61–v0.65.2"
```

---

## Task 2: Update Application Layer Docs (pi-coding-agent + extension system)

**Files:**
- Modify: `~/src/tui/pi-mono-docs/04-pi-coding-agent.md`
- Modify: `~/src/tui/pi-mono-docs/07-extension-system.md`

**Source to read:**
- `~/src/tui/pi-mono/packages/coding-agent/CHANGELOG.md` (full)
- `~/src/tui/pi-mono/packages/coding-agent/docs/sdk.md`
- `~/src/tui/pi-mono/packages/coding-agent/docs/extensions.md`
- `~/src/tui/pi-mono/packages/coding-agent/examples/sdk/13-session-runtime.ts`

- [ ] **Step 1: Read existing docs**

```bash
cat ~/src/tui/pi-mono-docs/04-pi-coding-agent.md
cat ~/src/tui/pi-mono-docs/07-extension-system.md
```

- [ ] **Step 2: Read source**

```bash
cat ~/src/tui/pi-mono/packages/coding-agent/CHANGELOG.md
cat ~/src/tui/pi-mono/packages/coding-agent/docs/sdk.md
cat ~/src/tui/pi-mono/packages/coding-agent/examples/sdk/13-session-runtime.ts
grep -n "defineTool\|AgentSessionRuntime\|createAgentSessionRuntime\|unified diagnostics\|session_directory" ~/src/tui/pi-mono/packages/coding-agent/docs/extensions.md | head -30
```

- [ ] **Step 3: Update 04-pi-coding-agent.md**

Update version header to v0.65.2. Apply changes:

**ModelRegistry section** — update constructor usage:

```markdown
### ModelRegistry Construction (v0.64.0+)

`ModelRegistry` no longer has a public constructor. Use:

```typescript
import { AuthStorage, ModelRegistry } from "@mariozechner/pi-coding-agent";

const authStorage = AuthStorage.create();

// File-backed (reads ~/.pi/models.json)
const modelRegistry = ModelRegistry.create(authStorage);

// In-memory (built-in models only, no file I/O)
const modelRegistry = ModelRegistry.inMemory(authStorage);
```
```

**AgentSessionRuntime section** (new for v0.65.0):

```markdown
## AgentSessionRuntime (v0.65.0+)

`AgentSessionRuntime` replaces the removed session-replacement methods on `AgentSession`. It takes a `CreateAgentSessionRuntimeFactory` closure that closes over process-global inputs and recreates cwd-bound services for each session switch.

```typescript
import {
  type CreateAgentSessionRuntimeFactory,
  createAgentSessionFromServices,
  createAgentSessionRuntime,
  createAgentSessionServices,
  getAgentDir,
  SessionManager,
} from "@mariozechner/pi-coding-agent";

const createRuntime: CreateAgentSessionRuntimeFactory = async ({
  cwd,
  sessionManager,
  sessionStartEvent,
}) => {
  const services = await createAgentSessionServices({ cwd });
  return {
    ...(await createAgentSessionFromServices({ services, sessionManager, sessionStartEvent })),
    services,
    diagnostics: services.diagnostics,
  };
};

const runtime = await createAgentSessionRuntime(createRuntime, {
  cwd: process.cwd(),
  agentDir: getAgentDir(),
  sessionManager: SessionManager.create(process.cwd()),
});

// Session operations (all use the same factory, rebuilding cwd-bound state)
await runtime.newSession();
await runtime.switchSession("/path/to/session.jsonl");
await runtime.fork("entry-id");

// After replacement: runtime.session is the new live session
// Rebind any session-local subscriptions after each switch
```

The factory pattern ensures that `/new`, `/resume`, `/fork`, import, and startup all use identical initialization logic with no special cases.
```

**defineTool section** (new for v0.65.0):

```markdown
## defineTool() Helper (v0.65.0+)

`defineTool()` creates standalone custom tool definitions with full TypeScript parameter type inference. No manual type casts needed:

```typescript
import { defineTool } from "@mariozechner/pi-coding-agent";
import { Type } from "@sinclair/typebox";

const greetTool = defineTool({
  name: "greet",
  description: "Greet someone by name",
  parameters: Type.Object({
    name: Type.String({ description: "Name to greet" }),
  }),
  execute: async ({ name }) => ({
    content: [{ type: "text", text: `Hello, ${name}!` }],
    isError: false,
  }),
});
```

Array literals of tool definitions also keep inferred types without manual casts. See [docs/extensions.md](https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/extensions.md) for examples.
```

**Unified diagnostics section** (new for v0.65.0):

```markdown
## Unified Diagnostics (v0.65.0+)

Arg parsing, service creation, session option resolution, and resource loading now return structured `info`/`warning`/`error` diagnostics instead of logging or exiting. The application layer decides presentation and exit behavior.

Unknown single-dash CLI flags (e.g., `-s`) now produce an `error` diagnostic instead of being silently ignored.
```

**Session changes** — update `/tree` docs for label timestamps:

```markdown
### /tree — Session Tree Navigation

Press `Shift+T` to toggle timestamp labels on tree entries. Timestamps use smart date formatting (relative for recent entries, absolute for older ones) and are preserved through branching.
```

- [ ] **Step 4: Update 07-extension-system.md**

Update version header. Apply changes:

**session_start event** — replace `session_switch`/`session_fork` with unified `session_start`:

```markdown
### session_start (v0.65.0+ unified API)

`session_switch` and `session_fork` events were removed in v0.65.0. Use `session_start` with `event.reason`:

```typescript
pi.on("session_start", async (event, ctx) => {
  // event.reason: "startup" | "reload" | "new" | "resume" | "fork"
  // event.previousSessionFile: present for "new", "resume", "fork"

  if (event.reason === "fork") {
    ctx.ui.notify(`Forked from ${event.previousSessionFile}`, "info");
  }
});
```

**Migration from session_switch / session_fork:**

```typescript
// Before (removed in v0.65.0)
pi.on("session_switch", async (event, ctx) => { ... });
pi.on("session_fork", async (_event, ctx) => { ... });

// After
pi.on("session_start", async (event, ctx) => {
  if (event.reason === "new" || event.reason === "resume" || event.reason === "fork") {
    // was session_switch or session_fork
  }
});
```
```

**ctx.signal section** (add v0.63.2 feature):

```markdown
### ctx.signal (v0.63.2+)

The current agent abort signal, or `undefined` when no agent turn is active. Use it to forward cancellation into nested model calls, fetch, and other abort-aware work:

```typescript
pi.on("tool_result", async (event, ctx) => {
  const response = await fetch("https://api.example.com/data", {
    signal: ctx.signal,
  });
  // ...
});
```
```

**prepareArguments section** (add v0.64.0 feature):

```markdown
### ToolDefinition.prepareArguments (v0.64.0+)

Attach to any tool definition to normalize raw model arguments before schema validation. Built-in `edit` tool uses this to transparently handle sessions created with the old single-edit schema:

```typescript
pi.registerTool({
  name: "my-tool",
  description: "...",
  parameters: MySchema,
  prepareArguments: (rawArgs) => {
    // Migrate old arg shape to new before validation
    if (rawArgs.oldParam) {
      return { newParam: rawArgs.oldParam };
    }
    return rawArgs;
  },
  execute: async (params, ctx) => { ... },
});
```
```

**setHiddenThinkingLabel section** (add v0.64.0 feature):

```markdown
### ctx.ui.setHiddenThinkingLabel() (v0.64.0+)

Customize the collapsed thinking block label in interactive mode:

```typescript
pi.on("session_start", async (_event, ctx) => {
  ctx.ui.setHiddenThinkingLabel("🧠 Reasoning hidden");
});
```

No-op in RPC mode. See `examples/extensions/hidden-thinking-label.ts` for a complete example.
```

**session_directory removal note:**

```markdown
> **Removed in v0.65.0:** `session_directory` has been removed from both the extension API and settings API.
```

- [ ] **Step 5: Verify fact-checks**

```bash
grep -rn "session_switch\|session_fork" ~/src/tui/pi-mono/packages/coding-agent/docs/ | head -10
grep -n "defineTool\|AgentSessionRuntime\|createAgentSessionRuntime" ~/src/tui/pi-mono/packages/coding-agent/src/ -r | head -20
grep -n "session_directory" ~/src/tui/pi-mono/packages/coding-agent/CHANGELOG.md | head -5
```

- [ ] **Step 6: Commit Task 2**

```bash
cd ~/src/tui/pi-mono-docs
git add 04-pi-coding-agent.md 07-extension-system.md
git commit -m "docs: update coding-agent and extension system for v0.63–v0.65.2"
```

---

## Task 3: Update Infrastructure Docs (pi-tui + sessions)

**Files:**
- Modify: `~/src/tui/pi-mono-docs/05-pi-tui-terminal-ui.md`
- Modify: `~/src/tui/pi-mono-docs/08-sessions-and-persistence.md`

**Source to read:**
- `~/src/tui/pi-mono/packages/tui/CHANGELOG.md` (full)
- `~/src/tui/pi-mono/packages/coding-agent/docs/session.md`
- `~/src/tui/pi-mono/packages/coding-agent/docs/tree.md`

- [ ] **Step 1: Read existing docs**

```bash
cat ~/src/tui/pi-mono-docs/05-pi-tui-terminal-ui.md
cat ~/src/tui/pi-mono-docs/08-sessions-and-persistence.md
```

- [ ] **Step 2: Read source**

```bash
cat ~/src/tui/pi-mono/packages/tui/CHANGELOG.md
cat ~/src/tui/pi-mono/packages/coding-agent/docs/session.md | head -100
cat ~/src/tui/pi-mono/packages/coding-agent/docs/tree.md 2>/dev/null | head -60
grep -n "PI_TUI_WRITE_LOG\|requestRender\|16ms\|frame budget" ~/src/tui/pi-mono/packages/tui/ -r | head -20
```

- [ ] **Step 3: Update 05-pi-tui-terminal-ui.md**

Update version header. Apply changes:

**Render scheduling section** (add v0.65.2):

```markdown
### Render Scheduling (v0.65.2+)

`requestRender()` calls are coalesced to a 16ms frame budget (≈ 60 fps) under heavy streaming output. Immediate renders via `requestRender(true)` bypass coalescing and render synchronously.
```

**Debug logging section** (add v0.63.0):

```markdown
### Debug Logging

Set `PI_TUI_WRITE_LOG` to a directory path to enable per-instance TUI logging. Each instance creates a unique file: `tui-<timestamp>-<pid>.log`. Useful when debugging multiple simultaneous pi sessions:

```bash
PI_TUI_WRITE_LOG=/tmp/pi-tui-logs pi
```
```

**Keyboard protocol section** — add v0.64.0 fixes:

```markdown
#### Kitty Keyboard Protocol Fixes (v0.64.0)

- Keypad functional keys now normalize to logical digits, symbols, and navigation keys, fixing numpad input in terminals such as iTerm2 that previously inserted Private Use Area characters.
- CSI cell size responses now only consume exact `CSI 6 ; height ; width t` replies, so bare `Escape` is no longer swallowed while waiting for terminal image metadata.
```

**Autocomplete section** — add v0.65.0 fix:

```markdown
#### Async Argument Completions (v0.65.0)

Slash-command argument autocomplete now awaits async `getArgumentCompletions()` results and ignores invalid return values, preventing crashes when extension commands provide asynchronous completions.
```

- [ ] **Step 4: Update 08-sessions-and-persistence.md**

Update version header. Apply changes:

**AgentSessionRuntime section** (new):

```markdown
## AgentSessionRuntime (v0.65.0+)

For SDK users who need to switch between sessions at runtime, `AgentSessionRuntime` provides factory-based session replacement. Unlike the removed mutator methods on `AgentSession`, the runtime ensures all cwd-bound services are rebuilt on every switch, `/new`, `/resume`, `/fork`, and import operation.

See **Task 2 (pi-coding-agent section)** for the full API reference and example code.
```

**Session cwd handling section** (update for v0.65.1):

```markdown
### Session Working Directory (v0.65.1 fix)

When resuming or importing a session whose original working directory no longer exists:
- **Interactive mode**: Prompts the user to continue in the current cwd.
- **Non-interactive mode**: Fails with a clear error.
```

**session_directory removal section**:

```markdown
> **Removed in v0.65.0:** The `session_directory` setting has been removed from both the extension API and settings API. Extensions that relied on this should use `ctx.sessionManager.getSessionFile()` to derive the session's base directory.
```

- [ ] **Step 5: Commit Task 3**

```bash
cd ~/src/tui/pi-mono-docs
git add 05-pi-tui-terminal-ui.md 08-sessions-and-persistence.md
git commit -m "docs: update pi-tui and sessions for v0.63–v0.65.2"
```

---

## Task 4: Update Version/Reference Docs

**Files:**
- Modify: `~/src/tui/pi-mono-docs/README.md`
- Modify: `~/src/tui/pi-mono-docs/01-architecture-overview.md`
- Modify: `~/src/tui/pi-mono-docs/09-version-history.md`

**Source to read:**
- `~/src/tui/pi-mono/packages/coding-agent/CHANGELOG.md` (full)
- `~/src/tui/pi-mono/packages/ai/CHANGELOG.md`
- `~/src/tui/pi-mono/packages/agent/CHANGELOG.md`
- `~/src/tui/pi-mono/packages/tui/CHANGELOG.md`

- [ ] **Step 1: Read existing docs**

```bash
cat ~/src/tui/pi-mono-docs/README.md
cat ~/src/tui/pi-mono-docs/01-architecture-overview.md
cat ~/src/tui/pi-mono-docs/09-version-history.md
```

- [ ] **Step 2: Update README.md**

Change `**Current release:** v0.60.0 (2026-03-19)` to:

```markdown
**Current release:** v0.65.2 (2026-04-06)
```

Update the Core Documentation table `(01)` description to note the version is now v0.65.2 and that post-v0.60 breaking changes are documented.

- [ ] **Step 3: Update 01-architecture-overview.md**

Change all occurrences of `v0.60.0` in the version header/intro to `v0.65.2`. Add a note in the versioning section about the release cadence since v0.60.0 (9 releases in ~3 weeks).

- [ ] **Step 4: Update 09-version-history.md**

Read existing version history to understand the era structure. Add a new era section after Era 6 for the v0.61–v0.65 releases. The era should follow the same style as existing eras with release dates, descriptions, and breaking change notes.

New releases to document (all dates from CHANGELOG files):

```markdown
## Era 7: Session Runtime & API Hardening (v0.61.0 – v0.65.2, 2026-03-20 to 2026-04-06)

### v0.61.0 (2026-03-20)
- **ai**: Added `gpt-5.4-mini` for `openai-codex` provider; `validateToolArguments` fallback for restricted runtimes; Google Vertex API key placeholder fix; OpenRouter `reasoning.effort` fix; Bedrock prompt caching for inference profiles

### v0.61.1 (2026-03-20)
- **ai**: Normalized MiniMax model metadata; added `MiniMax-M2.1-highspeed` entries
- **tui**: Fixed shared keybinding resolution (user overrides no longer evict unrelated default shortcuts); fixed Termux software keyboard height changes

### v0.62.0 (2026-03-23)
- **ai**: Added `BedrockOptions.requestMetadata` for AWS cost allocation tagging; exported `BedrockOptions` type; fixed various provider streaming issues
- **tui**: Fixed `truncateToWidth()` for very large strings; fixed markdown heading styling

### v0.63.0 (2026-03-27) **BREAKING**
- **ai**: Removed `ModelRegistry.getApiKey()` → use `getApiKeyAndHeaders()`; removed deprecated `minimax`/`minimax-cn` model IDs
- **tui**: Added `PI_TUI_WRITE_LOG` directory for per-instance debug logging; various TUI fixes

### v0.63.1 (2026-03-27)
- **ai**: Added `gemini-3.1-pro-preview-customtools` for `google-vertex`; Ollama overflow detection; Anthropic HTTP 413 overflow detection
- **coding-agent**: Fixed compaction duplicate issues, skill discovery, edit tool diff rendering, built-in tool override rendering

### v0.63.2 (2026-03-29)
- **agent**: Added `Agent.signal` for active abort signal
- **coding-agent**: `ctx.signal` in `ExtensionContext`; edit tool unified to `edits[]` schema only
- **tui**: Fixed TUI cell size response; Kitty keyboard protocol keypad normalization

### v0.64.0 (2026-03-29) **BREAKING**
- **ai**: Added faux provider helpers for deterministic tests (`registerFauxProvider`, `fauxAssistantMessage`, etc.)
- **agent**: Added `AgentTool.prepareArguments` hook
- **coding-agent**: `ModelRegistry` public constructor removed → `ModelRegistry.create()` / `ModelRegistry.inMemory()`; `ctx.ui.setHiddenThinkingLabel()`; `ToolDefinition.prepareArguments`; built-in `edit` uses `prepareArguments` for legacy schema compat
- **tui**: Fixed TUI cell size CSI response parsing; Kitty numpad normalization

### v0.65.0 (2026-04-03) **BREAKING**
- **agent**: `AgentState` reshaped (renamed fields, readonly enforcement, accessor properties); all `Agent` setXxx() mutator methods removed; `subscribe()` now async + receives `AbortSignal`
- **coding-agent**: Added `AgentSessionRuntime` / `createAgentSessionRuntime()`; added `defineTool()` helper; label timestamps in `/tree` (Shift+T); unified diagnostics model; removed `session_switch`/`session_fork` extension events; removed `session_directory`; unknown single-dash flags now error
- **ai**: Z.ai tool streaming; Bedrock throttling fix; various provider fixes

### v0.65.1 (2026-04-05)
- **coding-agent**: Bash output line-truncation fix (persists full output to temp file); RpcClient stderr forwarding; theme file watcher async error handling; session cwd missing-directory handling; resource collision precedence fix (project/user resources override package resources); `--mode json` piped stdin preservation; git/npm CLI extension path resolution; removed stale `/exit` docs
- **ai**: OpenRouter `cached_tokens` normalization; `cache_write_tokens` preservation

### v0.65.2 (2026-04-06)
- **tui**: Render scheduling coalesced to 16ms frame budget (throttle under streaming load)
- **coding-agent**: Earendil startup announcement with inline image; Anthropic subscription auth warning for third-party billing
- **ai**: Updated generated model catalog
```

Also update the **Breaking Changes Summary** table to add:

```markdown
- **v0.63.0**: `ModelRegistry.getApiKey()` removal, deprecated minimax IDs removed
- **v0.64.0**: `ModelRegistry` public constructor removal
- **v0.65.0**: `AgentState` reshape, Agent mutator removal, `session_switch`/`session_fork` removal, `session_directory` removal
```

And update **Notable Model Milestones** to add:
```markdown
| 2026-03-20 | v0.61.0 | gpt-5.4-mini for openai-codex |
| 2026-03-27 | v0.63.1 | gemini-3.1-pro-preview-customtools for google-vertex |
```

- [ ] **Step 5: Commit Task 4**

```bash
cd ~/src/tui/pi-mono-docs
git add README.md 01-architecture-overview.md 09-version-history.md
git commit -m "docs: update README, architecture overview, and version history to v0.65.2"
```

---

## Task 5: Update Supplementary Reference Docs

**Files:**
- Modify: `~/src/tui/pi-mono-docs/10-fact-check-report.md`
- Modify: `~/src/tui/pi-mono-docs/changelog.md`
- Modify: `~/src/tui/pi-mono-docs/extensions.md`
- Modify: `~/src/tui/pi-mono-docs/sessions.md`
- Modify: `~/src/tui/pi-mono-docs/cli-reference.md`
- Modify: `~/src/tui/pi-mono-docs/configuration.md`
- Modify: `~/src/tui/pi-mono-docs/providers.md`
- Modify: `~/src/tui/pi-mono-docs/tools.md`

**Source to read:**
- `~/src/tui/pi-mono/packages/coding-agent/docs/providers.md`
- `~/src/tui/pi-mono/packages/coding-agent/docs/settings.md`
- `~/src/tui/pi-mono/packages/coding-agent/docs/extensions.md`
- npm page: `https://www.npmjs.com/package/@mariozechner/pi-coding-agent`
- GitHub: check current star count with `gh api repos/badlogic/pi-mono --jq '.stargazers_count'`

- [ ] **Step 1: Read existing docs**

```bash
cat ~/src/tui/pi-mono-docs/10-fact-check-report.md
cat ~/src/tui/pi-mono-docs/changelog.md | head -50
cat ~/src/tui/pi-mono-docs/extensions.md | head -50
cat ~/src/tui/pi-mono-docs/cli-reference.md | head -50
```

- [ ] **Step 2: Fact-check current stats**

```bash
gh api repos/badlogic/pi-mono --jq '{"stars": .stargazers_count, "forks": .forks_count, "open_issues": .open_issues_count}'
npm view @mariozechner/pi-coding-agent --json 2>/dev/null | grep -E '"version"|"time"' | head -5
```

Also fetch: `https://www.npmjs.com/package/@mariozechner/pi-coding-agent` to check download counts

- [ ] **Step 3: Update 10-fact-check-report.md**

Update the report header to v0.65.2 and current date (2026-04-09). Update all stats that have changed (GitHub stars, npm version, last publish date). Add a verification note for each new breaking change that was confirmed against source code.

- [ ] **Step 4: Update changelog.md**

Add entries for v0.61.0–v0.65.2 in the same format as existing entries. Reference the detailed notes in 09-version-history.md.

- [ ] **Step 5: Update extensions.md**

Apply same changes as 07-extension-system.md:
- Replace `session_switch`/`session_fork` with `session_start` + `event.reason`
- Add `ctx.signal` section
- Add `prepareArguments` section
- Add `setHiddenThinkingLabel` section
- Add `defineTool` section
- Note `session_directory` removal

- [ ] **Step 6: Update sessions.md**

Add AgentSessionRuntime section and session cwd handling note from Task 3.

- [ ] **Step 7: Update cli-reference.md**

```markdown
**Changed behavior (v0.65.1+):**
- Unknown single-dash CLI flags (e.g., `-s`) now produce an error. Previously they were silently ignored.
- `--mode json` now preserved when input is piped via stdin (was falling back to plain text).
- `/exit` command: Removed. Use `q` or `Ctrl+C` to quit.

**New interactive commands:**
- `/tree` — Press `Shift+T` to toggle timestamps on tree entries (v0.65.0+).
```

- [ ] **Step 8: Update configuration.md**

Add note that `session_directory` was removed in v0.65.0. Update any ModelRegistry instantiation examples to use `ModelRegistry.create()`.

- [ ] **Step 9: Update providers.md**

Update model listings:
- Remove `minimax` and `minimax-cn` direct provider IDs (removed v0.63.0)
- Add `MiniMax-M2.7` and `MiniMax-M2.7-highspeed` as the supported MiniMax IDs
- Add `gemini-3.1-pro-preview-customtools` under `google-vertex`
- Add `gpt-5.4-mini` under `openai-codex`
- Update model catalog version note to v0.65.2

- [ ] **Step 10: Update tools.md**

```markdown
### edit tool (v0.63.2+ schema change)

The `edit` tool now accepts **only** the `edits[]` array shape. The legacy single-edit top-level `oldText`/`newText` schema was removed. Sessions created before v0.63.2 are transparently migrated via the `prepareArguments` hook.

```typescript
// Current schema (v0.63.2+)
{
  "path": "/src/file.ts",
  "edits": [
    { "oldText": "foo", "newText": "bar" },
    { "oldText": "baz", "newText": "qux" }
  ]
}
```

### bash tool (v0.65.1 fix)

Output truncated by line count (> 2000 lines) is now always persisted to a temp file in its entirety. The LLM receives a truncated view, but the full output remains accessible. This prevents data loss when output exceeds 2000 lines but stays under the byte threshold.
```

- [ ] **Step 11: Commit Task 5**

```bash
cd ~/src/tui/pi-mono-docs
git add 10-fact-check-report.md changelog.md extensions.md sessions.md cli-reference.md configuration.md providers.md tools.md
git commit -m "docs: update supplementary reference docs for v0.61–v0.65.2"
```

---

## Task 6: Update Package Reference Docs

**Files:**
- Modify: `~/src/tui/pi-mono-docs/packages/ai.md`
- Modify: `~/src/tui/pi-mono-docs/packages/agent.md`
- Modify: `~/src/tui/pi-mono-docs/packages/tui.md`
- Modify: `~/src/tui/pi-mono-docs/packages/mom.md`
- Modify: `~/src/tui/pi-mono-docs/packages/web-ui.md`
- Modify: `~/src/tui/pi-mono-docs/packages/pods.md`

**Source to read:**
- `~/src/tui/pi-mono/packages/ai/package.json` — check version and exports
- `~/src/tui/pi-mono/packages/agent/package.json`
- `~/src/tui/pi-mono/packages/tui/package.json`
- `~/src/tui/pi-mono/packages/mom/CHANGELOG.md`
- `~/src/tui/pi-mono/packages/web-ui/CHANGELOG.md`

- [ ] **Step 1: Read existing package docs**

```bash
cat ~/src/tui/pi-mono-docs/packages/ai.md
cat ~/src/tui/pi-mono-docs/packages/agent.md
cat ~/src/tui/pi-mono-docs/packages/tui.md
head -30 ~/src/tui/pi-mono-docs/packages/mom.md
head -30 ~/src/tui/pi-mono-docs/packages/web-ui.md
head -30 ~/src/tui/pi-mono-docs/packages/pods.md
```

- [ ] **Step 2: Check source for mom/web-ui/pods changes**

```bash
cat ~/src/tui/pi-mono/packages/mom/CHANGELOG.md | head -50
cat ~/src/tui/pi-mono/packages/web-ui/CHANGELOG.md 2>/dev/null | head -50
cat ~/src/tui/pi-mono/packages/pods/CHANGELOG.md 2>/dev/null | head -50
```

- [ ] **Step 3: Update packages/ai.md**

Apply same ModelRegistry changes from Task 1:
- `ModelRegistry.getApiKey()` → `getApiKeyAndHeaders()` (v0.63.0)
- `ModelRegistry` constructor → `ModelRegistry.create()` / `ModelRegistry.inMemory()` (v0.64.0)
- Add faux provider helpers (v0.64.0)
- Add `BedrockOptions.requestMetadata` (v0.62.0)
- Update version to v0.65.2

- [ ] **Step 4: Update packages/agent.md**

Apply same AgentState/Agent changes from Task 1:
- AgentState interface with new field names and readonly markers
- Mutator methods removal table
- `subscribe()` async signature
- `Agent.signal` (v0.63.2)
- `AgentTool.prepareArguments` (v0.64.0)
- Update version to v0.65.2

- [ ] **Step 5: Update packages/tui.md**

Apply same TUI changes from Task 3:
- Render scheduling (v0.65.2)
- `PI_TUI_WRITE_LOG` (v0.63.0)
- Keyboard fixes
- Update version to v0.65.2

- [ ] **Step 6: Update packages/mom.md, web-ui.md, pods.md**

For each: update version header to v0.65.2. Check CHANGELOG for any functional changes; add notes only if functional changes exist.

- [ ] **Step 7: Commit Task 6**

```bash
cd ~/src/tui/pi-mono-docs
git add packages/
git commit -m "docs: update package reference docs to v0.65.2"
```

---

## Task 7: Final Review and Push

**Files:** All modified files
**Source:** Current git state of `~/src/tui/pi-mono-docs`

- [ ] **Step 1: Verify all version references updated**

```bash
grep -rn "v0\.60\.0\|v0\.61\b\|v0\.62\b\|v0\.63\b\|v0\.64\b" ~/src/tui/pi-mono-docs/ --include="*.md" | grep -v "version-history\|fact-check\|changelog\|docs/superpowers" | head -20
```

Expected: Only legitimate historical references remain. If any `v0.60.0` appears in a doc header that should say `v0.65.2`, fix it.

- [ ] **Step 2: Verify no removed APIs are documented as current**

```bash
grep -rn "getApiKey\b\|session_switch\|session_fork\|session_directory\|streamMessage\b\|\.error\b\|setSystemPrompt\|setModel\b\|setThinkingLevel\|setTools\b\|replaceMessages\|clearMessages\|setToolExecution\|setBeforeToolCall\|setAfterToolCall\|setTransport\b\|setSteeringMode\|setFollowUpMode\|getSteeringMode\|getFollowUpMode" ~/src/tui/pi-mono-docs/ --include="*.md" | grep -v "version-history\|fact-check\|changelog\|migration\|Before\|removed\|was\|docs/superpowers" | head -20
```

Expected: Zero results (or only migration/historical context blocks).

- [ ] **Step 3: Final git status check**

```bash
cd ~/src/tui/pi-mono-docs
git log --oneline -8
git status
```

- [ ] **Step 4: Push to GitHub**

```bash
cd ~/src/tui/pi-mono-docs
git push origin main
```

Expected output: successful push to `https://github.com/mudrii/pi-mono-docs.git`

---

## Self-Review

### Spec Coverage Checklist

| Requirement | Task |
|-------------|------|
| Fetch latest from origin | Completed before plan (pulled to da6e9ea4) |
| Consult deepwiki docs | Tasks 1–3 (WebFetch deepwiki pages) |
| Review all files in ~/src/tui/pi-mono | Tasks 1–6 (source reads for each doc group) |
| Add full docs to ~/src/tui/pi-mono-docs | Tasks 1–6 |
| Focus on released versions only | All tasks: documented up to v0.65.2 tag only; unreleased skipped |
| Fact-check and validate | Task 5 (10-fact-check-report.md with npm/GitHub verification) |
| Check online for additional docs | Task 5 (npm, GitHub API, deepwiki) |
| Create git commit | Each task commits its slice |
| Push to GitHub | Task 7, Step 4 |
| Parallel execution | Tasks 1–6 designed for parallel dispatch |

### Placeholder Scan

No TBD, TODO, or "implement later" in the plan. All code blocks show exact API shapes. All bash commands are runnable.

### Type Consistency

`streamingMessage` (new name) used consistently throughout Tasks 1, 2, 4, 6. `errorMessage` (new name) used consistently. `getApiKeyAndHeaders()` used consistently. `session_start` with `event.reason` used consistently.
