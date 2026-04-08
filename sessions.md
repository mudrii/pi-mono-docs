# Sessions

Pi stores conversations as JSONL files in a tree structure, enabling branching, forking, sharing, and navigation.

---

## Session Storage

```
~/.pi/agent/sessions/
└── <session-id>/
    ├── session.jsonl     # Append-only conversation (one JSON object per line)
    └── metadata.json     # Title, timestamps, parent session reference
```

Override with: `PI_CODING_AGENT_DIR`. (The `session_directory` extension event was removed in v0.65.0.)

---

## Session Tree (Branching)

Sessions form a tree. Every `/fork` creates a child session that branches from the current point.

```
root-session
├── fork-1 (branch at message 5)
│   └── fork-1a (branch at message 8)
└── fork-2 (branch at message 12)
```

### Branching Commands

| Command | Action |
|---------|--------|
| `/fork [message]` | Create a child session from the current entry |
| `/tree` | View the full session tree with branch folding |
| `/resume` | Interactive session picker |

### `/tree` Navigation

- **Arrow keys** — navigate entries
- **Space** — fold/unfold a branch
- **Enter** — jump to selected entry
- **Segment jump** — digit keys (0–9) jump to labeled segments
- **b** — jump between bookmarks

---

## Session Picker (`/resume`)

Interactive TUI for browsing all sessions:

| Key | Action |
|-----|--------|
| `↑` / `↓` | Navigate list |
| `Enter` | Resume selected session |
| `Ctrl+R` | Open picker from anywhere |
| `Ctrl+N` | Named session filter |
| `Ctrl+P` | Toggle path display |
| `Ctrl+D` | Delete selected session |
| `r` | Rename session |

Sessions shown in threaded sort mode, revealing fork relationships.

---

## Session Labels and Bookmarks

Annotate sessions for easier navigation:

```
/label "Implemented authentication"
/bookmark "Before the database refactor"
/bookmark "key-checkpoint"
```

Labels appear in `/tree` and session pickers. Filter by label in `/resume`.

---

## Starting and Resuming

```bash
pi                          # Auto-resume last session
pi --new-session            # Always start fresh
pi --session <id>           # Resume specific session
pi -ne                      # Short form: new session
```

### Resume Keybinding Action

Configure `Ctrl+R` behavior in settings:

```json
{
  "keybindings": {
    "resume": "open-picker"    // "open-picker" | "new-session"
  }
}
```

---

## Context Compaction

When the conversation history approaches the model's context window limit, pi automatically compacts:

1. Older messages are summarized by the LLM
2. Summary replaces the verbose history
3. Recent messages are kept verbatim
4. Session continues seamlessly

**Manual compact:** `/compact`

**Check usage:** `/context`

**Resilience:** Auto-compaction handles API errors (e.g., 529 overload) by retrying with exponential backoff (v0.56.3+).

Configure:
```json
{
  "compaction": {
    "thresholdPercent": 80,
    "keepRecentMessages": 10
  }
}
```

---

## Session Sharing

### Export as HTML

```
/export
```

Generates a self-contained HTML file:
- Collapsible tool inputs and outputs
- Active path highlighting
- Branch/fork relationship visualization
- Embedded JSONL for download
- Uses your active theme

### Share to pi.dev

```
/share
```

Uploads to pi.dev (or your configured `PI_SHARE_VIEWER_URL`). Returns a shareable link.

Also supports sharing to GitHub Gists.

---

## AgentSessionRuntime (v0.65.0+)

For programmatic session switching in SDK integrations, use `AgentSessionRuntime`. It ensures cwd-bound services are rebuilt on every `/new`, `/resume`, `/fork`, or import operation. See `packages/coding-agent/docs/sdk.md` for details.

```typescript
// Session switching via runtime (replaces direct session.newSession() / session.switchSession())
await runtime.newSession();
await runtime.switchSession("/path/to/session.jsonl");
await runtime.fork("entry-id");
// After replacement, runtime.session is the new live session.
```

## Missing Working Directory (v0.65.1)

When resuming a session whose original working directory no longer exists, interactive mode prompts to continue in the current directory; non-interactive mode fails with a clear error.

## Session in Extensions

```typescript
// Programmatic session switching
await ctx.switchSession(sessionId);

// Get/set session name via RPC
// rpc: set_session_name
```

---

## JSONL Session Format

Each line is a JSON object. Message types:

```jsonl
{"role":"user","content":"Fix the authentication bug","timestamp":1234567890}
{"role":"assistant","content":[{"type":"text","text":"I'll look at the auth code..."}],"api":"anthropic-messages","provider":"anthropic","model":"claude-opus-4-6","usage":{"input":1234,"output":567,...},"stopReason":"toolUse","timestamp":1234567891}
{"role":"toolResult","toolCallId":"tool_abc","toolName":"read_file","content":[{"type":"text","text":"...file contents..."}],"isError":false,"timestamp":1234567892}
```

Custom extension message types can also appear in the JSONL.

---

## Multi-Session Workflow Example

```bash
# Start working on feature
pi --new-session
pi "Implement OAuth 2.0 login"

# Branch before a risky refactor
/fork "Before database schema change"

# Explore the risky path
/tree          # See the branch
/fork          # Fork again if needed

# If it works, continue on the fork
# If not, /resume → go back to the pre-fork session
```
