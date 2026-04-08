# CLI Reference

The `pi` binary is the main entry point for the pi coding agent.

---

## Synopsis

```
pi [FLAGS] [PROMPT]
```

If `PROMPT` is provided, pi runs in non-interactive (print) mode and exits. Otherwise, it starts the interactive TUI.

---

## Flags

| Flag | Description |
|------|-------------|
| `--model <ref>` | Select model at startup. Accepts exact ID, `provider/id`, fuzzy name, glob, or `name:thinking` (e.g., `sonnet:high`) |
| `--thinking <level>` | Thinking level: `off`, `minimal`, `low`, `medium`, `high`, `xhigh` |
| `--new-session` | Force a new session (don't resume last) |
| `--no-resume` | Alias for `--new-session` |
| `--session <id>` | Resume a specific session by ID |
| `--no-print` | In print mode, suppress model output to stdout |
| `--json` | Output JSONL event stream (for scripting) |
| `--rpc` | Run in RPC mode (LF-delimited JSONL over stdin/stdout) |
| `--offline` | Disable startup network operations |
| `--verbose` | Enable verbose logging |
| `-ne` | Alias for `--new-session` |
| `-ns` | Alias for `--no-resume` |
| `-np` | Alias for `--no-print` |

> **Note (v0.65.0):** Unknown single-dash flags now produce an error instead of being silently ignored. For example, `-s` will error. Use the full `--session` flag instead.

### `--mode json` with piped stdin (v0.65.1)

When input is piped via stdin, `--mode json` is now correctly preserved and JSONL output is maintained instead of falling back to plain text:

```bash
echo "Explain async/await" | pi --mode json
```

### Model Reference Formats

```bash
pi --model claude-opus-4-6            # Exact model ID
pi --model anthropic/claude-opus-4-6  # Provider-qualified
pi --model opus                        # Fuzzy match
pi --model '*sonnet*'                  # Glob pattern
pi --model sonnet:high                 # With thinking level
pi --model anthropic/opus:medium      # Provider + thinking
```

---

## Package Management

```bash
pi install <source>    # Install extension/skill/theme package
pi update [source]     # Update installed packages
pi list                # List installed packages
pi remove <package>    # Remove a package
pi uninstall           # Alias for uninstall
pi config              # Open TUI to enable/disable resources
```

**Source formats:**
```bash
pi install @scope/package-name       # npm package
pi install github.com/user/repo      # GitHub URL
pi install git@github.com:user/repo  # SSH URL
pi install ./local/path              # Local path
```

---

## Slash Commands (Interactive Mode)

Slash commands are available in the interactive TUI. Type `/` to see completions.

### Session Management

| Command | Description |
|---------|-------------|
| `/new` | Start a new session |
| `/resume` | Open session picker (browse, resume, rename, delete) |
| `/fork [message]` | Fork current session at this point |
| `/tree` | View session tree with branch folding and jump navigation; `Shift+T` toggles timestamps (v0.65.0) |
| `/label <text>` | Label current session entry |
| `/bookmark [label]` | Bookmark current position |
| `/export` | Export session as HTML (shareable) |
| `/share` | Share session to GitHub Gist or pi.dev |

### Navigation

| Command | Description |
|---------|-------------|
| `/up` | Navigate to previous session entry |
| `/down` | Navigate to next session entry |

### Model & Settings

| Command | Description |
|---------|-------------|
| `/model [ref]` | Switch model or open model picker |
| `/thinking [level]` | Set thinking level (`off`, `minimal`, ..., `xhigh`) |
| `/settings` | Open settings TUI editor |
| `/login` | OAuth login to a provider |
| `/logout` | Log out from a provider |

### Context

| Command | Description |
|---------|-------------|
| `/compact` | Manually trigger context compaction |
| `/context` | Show current context usage |
| `/clear` | Clear message history |

### Utility

| Command | Description |
|---------|-------------|
| `/reload` | Reload extensions without restarting |
| `/quit` | Exit pi |
| `/help` | Show available commands |

> **Note:** `/exit` was removed. Use `q` or `Ctrl+C` to quit the interactive TUI.

---

## Keyboard Shortcuts (TUI)

### Input / Editor

| Key | Action |
|-----|--------|
| `Enter` | Submit message (steers current work if agent is running) |
| `Alt+Enter` | Submit as follow-up (queued until agent completes) |
| `Shift+Enter` | Insert newline |
| `Ctrl+C` | Abort current agent run |
| `Ctrl+K` | Kill line to end (Emacs kill ring) |
| `Ctrl+Y` | Yank from kill ring |
| `Alt+Y` | Cycle kill ring entries |
| `Ctrl+Z` / `Ctrl+-` | Undo |
| `Ctrl+A` | Move to beginning of line |
| `Ctrl+E` | Move to end of line |
| `Ctrl+W` | Delete word before cursor |
| `Ctrl+B` / `Ctrl+F` | Move backward/forward one character (Emacs) |
| `Alt+Left` / `Alt+Right` | Move word backward/forward |
| `Ctrl+]` | Character jump navigation |
| `Tab` | File path autocomplete |

### Image Attachment

| Key | Action |
|-----|--------|
| `Alt+V` | Paste image from clipboard (Windows default) |
| `Ctrl+V` | Paste image (platform default) |

### Session Navigation

| Key | Action |
|-----|--------|
| `Ctrl+R` | Open session picker |
| `Ctrl+N` | Named session filter in picker |
| `0`–`9` | Digit keybindings (configurable) |

### Tool Output

| Key | Action |
|-----|--------|
| `Space` | Expand/collapse tool call |
| `e` | Expand all tool calls |
| `c` | Collapse all tool calls |

---

## Print Mode

Run pi non-interactively:

```bash
pi "Explain how async/await works in TypeScript"
# Output goes to stdout, pi exits when done

pi --json "Generate a JSON schema for a user object"
# Output as JSONL event stream

pi --no-print "Refactor src/utils.ts for readability"
# Run task silently (no stdout output)
```

### JSONL Event Stream (`--json`)

Each line is a JSON event:

```jsonl
{"type":"user_message","text":"What is 2+2?"}
{"type":"assistant_start"}
{"type":"text_delta","delta":"2"}
{"type":"text_delta","delta":" + 2 = 4"}
{"type":"assistant_end","stopReason":"stop"}
{"type":"token_usage","inputTokens":10,"outputTokens":5,"cost":0.0001}
```

---

## RPC Mode

For integration with editors, IDEs, or non-Node clients:

```bash
pi --rpc
```

Communicates via LF-delimited JSONL on stdin/stdout. Breaking change in v0.57.0: strict LF-only framing (no CRLF).

---

## Environment Variables

| Variable | Description |
|----------|-------------|
| `PI_CODING_AGENT_DIR` | Override agent directory (default: `~/.pi/agent`) |
| `PI_OFFLINE` | Disable network on startup (also: `--offline`) |
| `PI_CACHE_RETENTION` | Set to `long` for extended prompt caching |
| `PI_PACKAGE_DIR` | Package directory for Nix/Guix environments |
| `PI_SHARE_VIEWER_URL` | Custom share viewer URL |
| `PI_TUI_WRITE_LOG` | Path to capture raw ANSI stream for debugging |
| `PI_DEBUG_REDRAW` | Set to `1` to log redraw triggers |
| `PI_HARDWARE_CURSOR` | Set to `1` to enable hardware cursor |
| `PI_CLEAR_ON_SHRINK` | Set to `1` to clear screen on content shrink |
| `HTTP_PROXY` / `HTTPS_PROXY` | HTTP proxy for all LLM requests |
