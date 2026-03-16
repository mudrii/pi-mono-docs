# Architecture

Pi is a TypeScript monorepo (`pi-mono`) using npm workspaces. All packages share lockstep versioning.

---

## Three-Tier Package Hierarchy

```
Foundation Layer
└── @mariozechner/pi-ai
    Unified streaming API over 23+ LLM providers
    Model catalog, token/cost tracking, OAuth utilities
    9 distinct wire protocol implementations

Infrastructure Layer
├── @mariozechner/pi-agent-core
│   Stateless agentLoop() + stateful Agent class
│   Tool execution loop, lifecycle events
│   Parallel/sequential tool execution
│
└── @mariozechner/pi-tui
    Terminal UI with differential rendering
    Overlay system, editor, input, markdown rendering

Application Layer
├── @mariozechner/pi-coding-agent  (pi binary)
│   Interactive CLI, AgentSession, SessionManager
│   ExtensionRunner, SettingsManager, package management
│
├── @mariozechner/pi-web-ui
│   Browser web components for AI chat
│   IndexedDB storage, artifact system, tool renderers
│
├── @mariozechner/pi-mom
│   Slack bot, delegates to coding-agent core
│
└── @mariozechner/pi-pods
    CLI for vLLM GPU pod management
```

---

## Data Flow (Interactive Session)

```
User Input (terminal keyboard)
    ↓
TUI Input Handler (pi-tui)
    ↓
AgentSession (pi-coding-agent)
    ├── ExtensionRunner: fire lifecycle events
    ├── Context assembly: AGENTS.md + skills + system prompt
    └── Agent.prompt() (pi-agent-core)
         ↓
         agentLoop()
         ├── streamSimple() → pi-ai
         │   ├── Provider lookup by model.api
         │   ├── Message transformation (provider format)
         │   ├── HTTP/WebSocket request
         │   └── AssistantMessageEventStream (async iteration)
         │       ↓ text_delta events
         │       ↓ toolcall_end events
         │
         └── Tool Execution
             ├── beforeToolCall hook (extension can block)
             ├── Execute tool handler (parallel or sequential)
             │   ├── read_file
             │   ├── write_file
             │   ├── edit_file
             │   └── bash
             └── afterToolCall hook (extension can mutate result)
                 ↓
                 ToolResultMessage → next LLM turn
    ↓
AgentEvent stream → TUI rendering (pi-tui differential render)
```

---

## pi-ai: Provider Abstraction

### API Implementations (9 Wire Protocols)

| API ID | Protocol | Used By |
|--------|---------|---------|
| `openai-completions` | OpenAI Chat Completions | OpenAI, Groq, Cerebras, LM Studio, Ollama, OpenRouter, custom |
| `openai-responses` | OpenAI Responses API | OpenAI, openrouter |
| `openai-codex-responses` | Codex WebSocket | openai-codex |
| `anthropic-messages` | Anthropic Messages API | anthropic, github-copilot, kimi-for-coding |
| `bedrock-converse-stream` | AWS Bedrock Converse | amazon-bedrock |
| `google-generative-ai` | Google Generative AI | google |
| `google-gemini-cli` | Gemini CLI OAuth | google-gemini-cli |
| `google-vertex` | Vertex AI | google-vertex |
| `mistral-conversations` | Mistral native SDK | mistral |
| `azure-openai-responses` | Azure OpenAI Responses | azure-openai-responses |

### Model Registry

Models are stored in a generated `models.generated.ts` file (auto-generated from models.dev, OpenRouter, and Vercel AI Gateway APIs). Only tool-calling models are included.

Each model entry:
```typescript
{
  id: string,
  name: string,
  api: Api,           // Wire protocol
  provider: Provider, // Provider ID
  baseUrl: string,
  reasoning: boolean, // Supports thinking/reasoning
  input: ("text" | "image")[],
  cost: {
    input: number,    // $/million tokens
    output: number,
    cacheRead: number,
    cacheWrite: number
  },
  contextWindow: number,
  maxTokens: number
}
```

### Stream Event Protocol

```
AssistantMessageEventStream emits:
  start
  text_start / text_delta / text_end
  thinking_start / thinking_delta / thinking_end
  toolcall_start / toolcall_delta / toolcall_end
  done (reason: "stop" | "length" | "toolUse")
  error (reason: "aborted" | "error")
```

---

## pi-agent-core: Agent Loop

### Execution Modes

**Parallel tool execution (default since v0.58.0):**
```
LLM emits 3 tool calls
  → Run preflights sequentially (beforeToolCall hooks)
  → Execute all 3 tool handlers concurrently
  → Emit results in original source order
  → Build ToolResultMessages
  → Next LLM turn
```

**Sequential tool execution (opt-in):**
```
LLM emits 3 tool calls
  → Tool 1: preflight → execute → emit
  → Tool 2: preflight → execute → emit
  → Tool 3: preflight → execute → emit
  → Build ToolResultMessages
  → Next LLM turn
```

### Message Flow

```
Prompt → [UserMessage]
       → [AssistantMessage (with ToolCall)]
       → [ToolResultMessage(s)]
       → [AssistantMessage (final response)]
       → done
```

### Steering vs Follow-Up

**Steering** (interrupts current work):
- User presses Enter while agent is executing tools
- After current tool completes, remaining tools receive error results
- Steering message injected; LLM responds immediately

**Follow-up** (appended after completion):
- User presses Alt+Enter while agent is running
- Queued until agent has no more tool calls and no more steering
- Then injected as next turn's user message

---

## pi-coding-agent: Session Architecture

### Session Storage (JSONL Tree)

```
~/.pi/agent/sessions/
└── <session-id>/
    ├── session.jsonl       # Append-only conversation history
    └── metadata.json       # Title, timestamps, parent reference
```

Each line in `session.jsonl` is one of:
- `UserMessage`
- `AssistantMessage`
- `ToolResultMessage`
- Custom extension message types

Sessions form a tree: `/fork` creates a new session branching from the current entry. `/tree` displays the full tree with folding.

### Extension System

```typescript
// Extension API lifecycle
pi.on("init", async (ctx) => {
  // Register tools, commands, hooks
  ctx.registerTool({ name, description, parameters, execute });
  ctx.registerCommand({ name, description, execute });
  ctx.on("session_start", handler);
  ctx.on("before_provider_request", interceptor);
});
```

Extensions are loaded from:
1. Built-in tools (`read`, `write`, `edit`, `bash`)
2. `~/.pi/agent/extensions/` (user-global)
3. `.pi/extensions/` (project-level)
4. Installed packages (`pi install`)

---

## pi-tui: Rendering Architecture

### Differential Rendering (3 Strategies)

| Scenario | Strategy |
|----------|----------|
| First render | Output all lines without clearing scrollback |
| Width change or change above viewport | Full screen clear + re-render |
| Normal update | Move cursor to first changed line, clear to end, render changes |

All rendering is wrapped in synchronized output escape sequences (CSI `?2026h/l`) for atomic, near-flicker-free updates.

### Component Model

```
TUI
├── Container (root)
│   ├── Component[] (children)
│   └── Component[] (overlays)
└── Terminal interface
    ├── ProcessTerminal (stdin/stdout)
    └── VirtualTerminal (testing)
```

Components implement:
- `render(width: number): string[]` — return lines to display
- `handleInput?(data: Buffer): boolean` — handle keyboard input
- `invalidate?(): void` — mark for redraw

---

## Cross-Provider Context Handoff

When switching providers mid-session, thinking traces are converted:

- Anthropic `ThinkingContent` → `<thinking>` XML in user message (for OpenAI)
- OpenAI reasoning items → preserved with `thinkingSignature` for replay
- Redacted thinking (safety filter) → preserved as-is, flagged with `redacted: true`

This maintains session continuity when switching between fundamentally different provider architectures.

---

## Cost Tracking

Costs are calculated automatically from model metadata:

```
total_cost = (input_tokens / 1,000,000) × model.cost.input
           + (output_tokens / 1,000,000) × model.cost.output
           + (cache_read_tokens / 1,000,000) × model.cost.cacheRead
           + (cache_write_tokens / 1,000,000) × model.cost.cacheWrite
```

Displayed in the TUI footer. Click to view detailed breakdown.

---

## TypeScript Configuration

All packages extend `tsconfig.base.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "strict": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  }
}
```

Build order (enforced by monorepo build script):
`tui → ai → agent → coding-agent → mom → web-ui → pods`
