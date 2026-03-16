# Context Engineering

Pi gives you precise, transparent control over what goes into the LLM context. Nothing is injected silently.

---

## System Prompt Architecture

Pi's system prompt has three layers:

| Layer | Source | Effect |
|-------|--------|--------|
| Default system prompt | Built-in (~1,000 tokens) | Core agent instructions |
| `APPEND_SYSTEM.md` | `~/.pi/agent/` or `.pi/` | Appended to default |
| `SYSTEM.md` | `~/.pi/agent/` or `.pi/` | **Replaces** the default entirely |

Project files (`.pi/`) take precedence over user-global (`~/.pi/agent/`).

---

## AGENTS.md — Project Instructions

`AGENTS.md` files provide project-specific instructions to the agent. They are discovered automatically:

**Discovery order:**
1. `cwd/AGENTS.md`
2. Parent directory `AGENTS.md`
3. ... up to the git repository root

All found files are concatenated, outermost to innermost. This allows repo-wide rules in the root and component-specific rules in subdirectories.

### Example AGENTS.md

```markdown
# Project: MyApp

## Technology Stack
- TypeScript + React frontend
- Node.js + Express backend
- PostgreSQL database
- Jest for tests

## Code Conventions
- Use `async/await`, never raw Promises
- All database queries go through the repository layer
- Tests must cover the happy path and at least one error case

## Do Not
- Run `npm install` in CI (use `npm ci`)
- Commit `.env` files
- Use `any` types

## Useful Commands
- `npm run dev` — start development server
- `npm test` — run all tests
- `npm run db:migrate` — run pending migrations
```

---

## SYSTEM.md — Custom System Prompt

Replace the default system prompt entirely:

```markdown
# ~/.pi/agent/SYSTEM.md
You are a senior TypeScript developer focusing on clean architecture.
You prefer explicit types over inference, small pure functions over classes,
and always write tests alongside code.

Working directory: {{cwd}}
Date: {{date}}
```

Available template variables:
- `{{cwd}}` — current working directory
- `{{date}}` — current date (ISO format)

---

## APPEND_SYSTEM.md — Extend Default Prompt

Add to the default system prompt without replacing it:

```markdown
# ~/.pi/agent/APPEND_SYSTEM.md
## Project-Specific Rules
Always prefer `const` over `let`. Never use `var`.
When editing TypeScript files, ensure the build still passes after changes.
```

---

## Skills

Skills are reusable Markdown prompt files discovered automatically:

```
.agents/skills/        # Project-level (auto-discovered since v0.54.0)
.pi/skills/            # Project-level (pi-specific)
~/.pi/skills/          # User-global
~/.agents/skills/      # User-global (shared with other agents)
```

### Skill Format

```markdown
---
name: code-review
description: Thorough code review checklist and procedure
tags: [review, quality]
---

# Code Review Procedure

## Checklist
1. Check for security vulnerabilities
2. Verify error handling
3. Ensure tests cover edge cases
4. Check performance implications

## Output Format
Provide a structured review with: Summary, Issues (severity, location, description), Suggestions.
```

### Using Skills

Skills are available via `/` autocomplete in the editor. Select one to inject it into your message.

Skills can also be explicitly mentioned:

```
Use the code-review skill to review src/auth/login.ts
```

---

## Prompt Templates

Prompt templates are reusable messages invokable as slash commands:

```
~/.pi/agent/prompts/standup.md   → /standup
.pi/prompts/deploy-checklist.md  → /deploy-checklist
```

### Template Format

```markdown
# Daily Standup

Please generate a standup report based on recent git commits and file changes.

## Format
- **Yesterday:** What was completed
- **Today:** What I plan to work on
- **Blockers:** Any impediments
```

### Invoking with Arguments

```
/standup for the auth module
```

Arguments are appended to the template content.

---

## @-Mentions and File Attachment

Reference files directly in your message:

```
@src/auth/login.ts can you refactor this to use async/await?
```

Pi reads the file and includes its content in the user message.

**Image attachment:**

```
Attach a screenshot then:
/    → select file via Tab completion
Alt+V (Windows) or Ctrl+V → paste from clipboard
```

Images are sent as base64-encoded content blocks. Requires a vision-capable model.

---

## Context Compaction

When conversation history grows large, pi compacts older messages:

### Automatic Compaction

Triggered at ~80% context window usage:
1. Older messages are summarized by the LLM
2. Summary injected as a special context message
3. Recent messages kept verbatim

### Manual Compaction

```
/compact
```

### Context Usage

```
/context
```

Shows: `Used: 45,234 / 200,000 tokens (22.6%)`

### From Extensions

```typescript
await ctx.compact();
const { used, total } = ctx.getContextUsage();
```

---

## Message Queuing

### Steering (Alt+Enter during run)

Press Enter while the agent is working to interrupt:
- Current tool completes
- Remaining queued tools receive error results
- Your message is injected immediately
- LLM responds to the steering message

### Follow-up (Alt+Enter)

Queue a message to be sent after the agent finishes:
- Agent completes all current work
- Follow-up message injected
- Agent runs another turn

Multiple follow-ups can be queued. They process one at a time by default.

---

## Context Transparency

Pi never injects context behind your back. Everything in the prompt is:

1. Your `AGENTS.md` files
2. Your `SYSTEM.md` or `APPEND_SYSTEM.md`
3. The conversation history
4. Any `@`-mentioned files
5. Tool definitions (for the current toolset)

You can inspect the full payload before it's sent using the `before_provider_request` extension hook.

---

## Resource Precedence (v0.55.0+)

When the same resource name exists at multiple levels:

1. Project (`.pi/`) takes precedence over user-global (`~/.pi/agent/`)
2. Within a level, extension-provided resources can be overridden by user files

Extension conflicts (two extensions providing same resource name) no longer unload extensions; the conflict is logged and the earlier-loaded resource wins.
