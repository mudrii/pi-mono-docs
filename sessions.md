# Sessions

Pi stores conversations as JSONL files in a tree structure, enabling branching, forking, sharing, and navigation.

---

## Session Storage

```
~/.pi/agent/sessions/
└── <encoded-cwd>/
    └── <timestamp>_<uuid>.jsonl  # Append-only tree session, one JSON object per line
```

Override with:
- `PI_CODING_AGENT_DIR=/path/to/dir` — relocate the entire agent dir (auth, settings, sessions, etc.).
- `PI_CODING_AGENT_SESSION_DIR=/path/to/sessions` — relocate sessions only (v0.71.0).
- `--session-dir <dir>` — per-invocation session directory.

(The `session_directory` extension event was removed in v0.65.0.)

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
pi                          # Normal interactive startup
pi --continue               # Continue previous session
pi --resume                 # Pick a session interactively
pi --session <id>           # Resume specific session
pi --fork <id>              # Fork specific session into a new session
pi --no-session             # Ephemeral session, not saved
```

### Customizing Resume Keys

`Ctrl+R` (open picker), `Ctrl+N` (named filter), and other session keys are remappable via `~/.pi/agent/keybindings.json` under the namespaced action IDs (`app.session.resume`, `app.session.tree`, `app.session.new`, etc.). See [configuration.md](configuration.md) and `packages/coding-agent/docs/keybindings.md` for the full action list.

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

Configure (defaults shown):

```json
{
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  }
}
```

- `reserveTokens` — tokens reserved for the LLM response (compaction triggers when input + reserve approaches the context window).
- `keepRecentTokens` — recent tokens that are *not* summarized (kept verbatim).
- Set `enabled: false` to disable auto-compaction entirely (use `/compact` manually).

See `packages/coding-agent/docs/compaction.md` for the full reference.

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
{"role":"assistant","content":[{"type":"text","text":"I'll look at the auth code..."}],"api":"anthropic-messages","provider":"anthropic","model":"claude-opus-4-7","usage":{"input":1234,"output":567,...},"stopReason":"toolUse","timestamp":1234567891}
{"role":"toolResult","toolCallId":"tool_abc","toolName":"read_file","content":[{"type":"text","text":"...file contents..."}],"isError":false,"timestamp":1234567892}
```

Custom extension message types can also appear in the JSONL.

---

## Multi-Session Workflow Example

```bash
# Start working on feature
pi
# or run non-interactively:
pi -p "Implement OAuth 2.0 login"

# Branch before a risky refactor
/fork "Before database schema change"

# Explore the risky path
/tree          # See the branch
/fork          # Fork again if needed

# If it works, continue on the fork
# If not, /resume → go back to the pre-fork session
```
