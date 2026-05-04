> **REMOVED (April 2026):** The `pi-mom` package (`@mariozechner/pi-mom`) was removed from the pi-mono monorepo as of April 30, 2026. It is no longer maintained or published. The information below is archived for historical reference only.

# @mariozechner/pi-mom

**Version:** 0.69.0 · **License:** MIT · [npm](https://www.npmjs.com/package/@mariozechner/pi-mom)

A self-managing Slack bot powered by Claude. Mom responds to @mentions and direct messages, executes bash commands, reads/writes files, and builds her own custom tools — all from within Slack.

---

## What Mom Does

Mom is a **coding agent in Slack**. She starts bare and builds what she needs:

- Responds to @mentions in channels and direct messages
- Executes shell commands (in Docker sandbox or on host)
- Reads/writes files in her workspace
- Installs packages (`apk`, `npm`, `pip`) as needed
- Creates custom CLI tools (skills) to extend her own capabilities
- Maintains persistent memory via `MEMORY.md` files
- Schedules recurring or one-shot tasks via an event system
- Uploads files back to Slack

---

## Installation

```bash
npm install -g @mariozechner/pi-mom
```

---

## Setup

### Create a Slack App

1. Go to [api.slack.com/apps](https://api.slack.com/apps) and create a new app
2. Enable **Socket Mode** (Settings → Socket Mode)
3. Generate an **App-Level Token** with `connections:write` scope — this is your `MOM_SLACK_APP_TOKEN`
4. Enable **Event Subscriptions** and subscribe to:
   - `app_mention`
   - `message.channels`
   - `message.groups`
   - `message.im`

5. Add **Bot Token Scopes**:
   - `app_mentions:read`
   - `channels:history`, `channels:read`
   - `chat:write`
   - `files:read`, `files:write`
   - `groups:history`, `groups:read`
   - `im:history`, `im:read`, `im:write`
   - `users:read`

6. Install the app to your workspace — this gives you the `MOM_SLACK_BOT_TOKEN`

### Configure Environment

```bash
export MOM_SLACK_APP_TOKEN=xapp-1-...
export MOM_SLACK_BOT_TOKEN=xoxb-...
export ANTHROPIC_API_KEY=sk-ant-...  # Optional: falls back to OAuth
```

### Run

```bash
# Recommended: Docker sandbox
mom --sandbox=docker:mom-sandbox ./data

# Host sandbox (not recommended — full system access)
mom --sandbox=host ./data

# Download channel history as plain text
mom --download <channel-id> ./data
```

---

## Workspace Layout

Mom organizes all data under her working directory:

```
./data/
├── MEMORY.md                    # Global memory (visible to all channels)
├── settings.json                # Runtime configuration
├── skills/                      # Global custom CLI tools
│   └── <skill-name>/
│       ├── SKILL.md             # Name + description frontmatter
│       └── *.sh / *.js / ...    # Tool implementation
│
├── events/                      # Event JSON files (file-watched)
│   ├── reminder-2026-03-15.json
│   └── standup.json
│
└── <CHANNEL_ID>/                # Per-channel directories
    ├── MEMORY.md                # Channel-specific memory
    ├── log.jsonl                # Message history (source of truth)
    ├── context.jsonl            # LLM context (synced from log)
    ├── attachments/             # Files shared by users
    ├── scratch/                 # Mom's working directory
    └── skills/                  # Channel-specific tools
```

---

## Tools

Mom has five built-in tools:

| Tool | Description |
|------|-------------|
| `bash` | Execute shell commands (output truncated at 10MB) |
| `read` | Read file contents (up to 2,000 lines or 50KB) |
| `write` | Create or overwrite files |
| `edit` | Surgical line-based file edits |
| `attach` | Upload a file to Slack |

Bash commands run inside the configured sandbox (Docker or host).

---

## Memory System

Mom maintains memory in Markdown files:

| File | Scope | Purpose |
|------|-------|---------|
| `data/MEMORY.md` | Global | Shared context across all channels |
| `data/<CHANNEL_ID>/MEMORY.md` | Per-channel | Channel-specific preferences, notes |

Both are loaded at the start of each conversation turn. Mom can update them freely using `write` or `edit`.

---

## Skills (Custom CLI Tools)

Skills are CLI tools that Mom creates herself to extend her capabilities. They live in:

- `data/skills/` — global, available in all channels
- `data/<CHANNEL_ID>/skills/` — channel-specific

### Skill Format

```
data/skills/my-tool/
├── SKILL.md          # Frontmatter with name + description
└── run.sh            # Implementation
```

**SKILL.md frontmatter:**

```markdown
---
name: my-tool
description: Does something useful
---

# My Tool

Additional documentation here.
```

Skills are discovered automatically and listed in Mom's system prompt so she knows what she can call.

### Creating a Skill

Mom can create skills herself using `write`:

```
You: Can you create a skill that fetches our Jira ticket summary?

Mom: [creates data/skills/jira-summary/SKILL.md and run.sh]
     Done! I've created a "jira-summary" skill. I'll use it next time you ask for ticket updates.
```

---

## Event System

Mom supports scheduled wake-ups via JSON event files in `data/events/`.

### Immediate Event

Triggers instantly when written:

```json
{
  "type": "immediate",
  "channel": "C1234567890",
  "message": "Deploy is done — please run smoke tests."
}
```

### One-Shot Event

Triggers at a specific time:

```json
{
  "type": "once",
  "channel": "C1234567890",
  "at": "2026-03-15T09:00:00+01:00",
  "message": "Daily standup reminder: post your update."
}
```

### Periodic Event

Recurring via cron schedule:

```json
{
  "type": "periodic",
  "channel": "C1234567890",
  "cron": "0 9 * * 1-5",
  "timezone": "Europe/Vienna",
  "message": "Good morning! Time for the daily standup."
}
```

**Silent completion:** Mom can respond with `[SILENT]` to suppress posting for periodic events when there's nothing worth saying.

**Limits:** Maximum 5 events queued per channel to prevent flooding.

Mom can create and manage event files herself using `write` and `edit`.

---

## Sandbox Modes

### Docker Sandbox (Recommended)

```bash
mom --sandbox=docker:mom-sandbox ./data
```

Mom runs all bash commands inside an Alpine Linux container named `mom-sandbox`. Docker must be installed and running. The sandbox provides:
- Isolation from the host system
- Controlled package installation (`apk add`)
- Persistent container across sessions

### Host Sandbox

```bash
mom --sandbox=host ./data
```

Commands run directly on the host. Gives Mom full system access — only use in trusted environments or with appropriate restrictions.

---

## Session and Context Management

### Message Storage

- **`log.jsonl`** — Source of truth: one JSON object per message, human-readable, grep-friendly
- **`context.jsonl`** — LLM context: synced from log, same format as pi coding-agent sessions

### Automatic Compaction

When conversation history approaches the model's context limit, Mom automatically compacts:
1. Summarizes older messages
2. Replaces them with a summary
3. Keeps recent messages verbatim
4. Continues without interruption

Configure in `data/settings.json`:
```json
{
  "compaction": {
    "thresholdPercent": 80,
    "keepRecentMessages": 10
  }
}
```

### Backfill on Startup

When Mom starts, she fetches and processes recent Slack history so she has context from before she was running.

---

## Model and Auth

Mom uses **Claude Sonnet 4.5** (Anthropic API) for all conversations.

**Authentication:**
1. Set `ANTHROPIC_API_KEY` environment variable, or
2. Use OAuth: mom inherits the pi OAuth flow from `~/.pi/agent/auth.json`

---

## Message Formatting

Mom uses Slack's **mrkdwn** format (not Markdown):

| Slack mrkdwn | Effect |
|-------------|--------|
| `*bold*` | Bold |
| `_italic_` | Italic |
| `` `code` `` | Inline code |
| ` ```code block``` ` | Code block |
| `<URL\|text>` | Link |
| `@user` → `<@USERID>` | User mention |
| `#channel` → `<#CHANID>` | Channel mention |

Tool execution details are posted in **Slack threads** to keep the main conversation clean.

---

## Multiple Channels

Mom handles multiple channels independently and concurrently:

- Each channel has its own directory, memory, context, and skills
- Messages are processed sequentially per channel (via queue) to prevent race conditions
- Multiple channels process in parallel across goroutines

---

## System Prompt

Mom's system prompt is dynamically generated per conversation and includes:

- Channel ID and user ID mappings
- Slack formatting rules
- Workspace layout documentation
- Discovered skill descriptions
- Events system overview
- Memory file locations
- Current timezone and date

The prompt is rebuilt on each turn to reflect the current workspace state.
