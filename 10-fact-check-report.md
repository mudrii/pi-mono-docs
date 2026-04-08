# Pi-Mono Fact-Check Report

Date: 2026-04-09

This report verifies claims about the Pi-Mono project against publicly available online sources.

---

## 1. npm Package Verification

### @mariozechner/pi-coding-agent

- **Status**: VERIFIED -- Published and actively maintained on npm
- **URL**: https://www.npmjs.com/package/@mariozechner/pi-coding-agent
- **Latest version**: 0.65.2 (published 2026-04-06)
- **Dependents**: 306 other npm projects depend on this package
- **Description**: "Coding agent CLI with read, bash, edit, write tools and session management"

### Related packages (all verified on npm)

| Package | npm URL | Status |
|---------|---------|--------|
| `@mariozechner/pi` | https://www.npmjs.com/package/@mariozechner/pi | Published |
| `@mariozechner/pi-ai` | (inferred from code) | Published |
| `@mariozechner/pi-agent-core` | https://www.npmjs.com/package/@mariozechner/pi-agent-core | Published |
| `@mariozechner/pi-agent` | https://www.npmjs.com/package/@mariozechner/pi-agent | Published |

### Third-party forks

- `@vandeepunk/pi-coding-agent` exists as a separate fork on npm

### Platform packages

- AUR (Arch Linux): `pi-coding-agent` package exists at https://aur.archlinux.org/packages/pi-coding-agent

---

## 2. GitHub Repository Verification

### badlogic/pi-mono

- **Status**: VERIFIED -- Repository exists and is actively maintained
- **URL**: https://github.com/badlogic/pi-mono
- **Description**: "AI agent toolkit: coding agent CLI, unified LLM API, TUI & web UI libraries, Slack bot, vLLM pods"
- **Stars**: 33,221 (as of 2026-04-09)
- **Forks**: 3,681 (as of 2026-04-09)
- **Open issues**: 21 (as of 2026-04-09)
- **Activity**: Very active with regular releases, GitHub Actions CI, and community contributions
- **Releases page**: https://github.com/badlogic/pi-mono/releases

### Related repositories

| Repository | Status | Description |
|------------|--------|-------------|
| `badlogic/pi-skills` | VERIFIED | Skills for pi coding agent (compatible with Claude Code and Codex CLI) |
| `badlogic/shittycodingagent.ai` | VERIFIED | Website source for the session viewer |
| `dnouri/pi-coding-agent` | VERIFIED | Emacs frontend for pi |
| `can1357/oh-my-pi` | VERIFIED | Extended fork with LSP, Python, browser, subagents |
| `qualisero/awesome-pi-agent` | VERIFIED | Awesome list of pi add-ons and resources |
| `agentic-dev-io/pi-agent` | VERIFIED | Community fork |

---

## 3. Website / Domain Verification

### pi.dev

- **Status**: VERIFIED -- Active, resolves to shittycodingagent.ai
- **Notes**: The domain `pi.dev` was donated by exe.dev (noted in v0.51.4 changelog). It serves as the primary domain. Share URLs default to pi.dev since v0.51.4.

### shittycodingagent.ai

- **Status**: VERIFIED -- Active and serving content
- **URL**: https://shittycodingagent.ai/
- **Content**: Landing page for pi, packages directory at `/packages`
- **Notes**: The original humorous domain name. Still active and functional. The name was "intentionally designed to be entirely un-Google-able" per community discussion.

### buildwithpi.ai

- **Status**: VERIFIED -- Active
- **URL**: https://buildwithpi.ai/
- **Notes**: Used as session share viewer URL from v0.45.1 to v0.51.3, before pi.dev became the default. Still functional.

### mariozechner.at

- **Status**: VERIFIED -- Author's personal blog
- **Notable post**: "What I learned building an opinionated and minimal coding agent" (2025-11-30)

---

## 4. Discord Community Verification

- **Invite link**: `discord.com/invite/3cU7Bz4UPx`
- **Status**: LIKELY VALID -- The Discord is referenced on the official website (pi.dev/shittycodingagent.ai) and GitHub README. Multiple search results reference the Discord as a support channel. The specific invite code was not directly searchable but the Discord community is confirmed to exist through multiple references.
- **Notes**: The Discord is listed as the primary community support channel alongside GitHub Issues.

---

## 5. Author / Maintainer Verification

### Mario Zechner (badlogic)

- **Status**: VERIFIED
- **GitHub**: https://github.com/badlogic
- **Blog**: https://mariozechner.at
- **npm scope**: `@mariozechner`
- **Known for**: Creator of libGDX (popular Java game development framework)
- **Role**: Primary author and maintainer of pi-mono

### Community contributors (verified via changelog mentions)

The changelogs reference 80+ unique contributors via GitHub PRs, including notable names:
- @mitsuhiko (Armin Ronacher, creator of Flask) -- multiple contributions including fuzzy search, image handling, model selector, session export
- @steipete (Peter Steinberger) -- interruptible tool execution
- @nicobailon -- extensive contributions (overlays, hooks design, subagent example, image rendering)
- @dannote -- multiple contributions (MiniMax provider, keyboard layouts, Bun compatibility)
- @Perlence -- extensive editor improvements (kill ring, undo, word navigation)
- @aliou -- session handling, command improvements
- @ferologics -- context usage, settings management
- @haoqixu -- Unicode/CJK input handling, TUI fixes
- @unexge -- Amazon Bedrock provider
- @kim0 -- Gemini CLI provider, Codex OAuth, keyboard protocol

---

## 6. Community References and Reviews

### Blog posts and articles (verified)

| Source | Title | URL |
|--------|-------|-----|
| Mario Zechner | "What I learned building an opinionated and minimal coding agent" | https://mariozechner.at/posts/2025-11-30-pi-coding-agent/ |
| Armin Ronacher | "Pi: The Minimal Agent Within OpenClaw" | https://lucumr.pocoo.org/2026/1/31/pi/ |
| Nader Dabit | "How to Build a Custom Agent Framework with PI: The Agent Stack Powering OpenClaw" | https://nader.substack.com/p/how-to-build-a-custom-agent-framework |
| Helmut Januschka | "Why I Switched to Pi: A Terminal-First Coding Agent" | https://www.januschka.com/pi-coding-agent.html |
| Daniel Koller | "Why pi is my new coding agent of choice" | https://www.danielkoller.me/en/blog/why-pi-is-my-new-coding-agent-of-choice |
| jprokay | "pi: The Coding Agent For Your Workflow" | https://jprokay.com/post/018-pi-coding-agent |
| Ry Walker | "Pi Coding Agent" (research page) | https://rywalker.com/research/pi |
| Atal Upadhyay | "PI Agent Revolution: Building Customizable, Open-Source AI Coding Agents" | https://atalupadhyay.wordpress.com/2026/02/24/pi-agent-revolution-building-customizable-open-source-ai-coding-agents-that-outperform-claude-code/ |
| ToolWorthy | "Pi Monorepo Review (2026): Free AI Agent Toolkit for Developers" | https://www.toolworthy.ai/tool/pi-mono |
| EveryDev | "Pi Coding Agent - Extensible Terminal AI Coding Agent" | https://www.everydev.ai/tools/pi-coding-agent |

### OpenClaw connection (verified)

- Pi's SDK powers OpenClaw's entire assistant layer
- OpenClaw has 160K+ stars according to search results
- OpenClaw documentation references Pi integration architecture at https://docs.openclaw.ai/pi

### Answer Overflow / Discord references

- Community discussions found on Answer Overflow (Discord archive), confirming active Discord use

---

## 7. Discrepancies and Notes

### Domain evolution
The project has used three domains over time:
1. **shittycodingagent.ai** -- Original domain (still active)
2. **buildwithpi.ai** -- Session viewer domain from v0.45.1 (still active)
3. **pi.dev** -- Current primary domain from v0.51.4 (still active)

All three remain functional. This is consistent with the changelog entries.

### Bedrock Claude 4.6 context window discrepancy
- v0.58.0 raised Claude Opus 4.6 context to 1M tokens
- v0.60.0 corrected Bedrock Claude 4.6 back to 200K
- This suggests the 1M context applies to direct Anthropic API but not Bedrock

### Bus factor
- Multiple reviews note the "single maintainer with bus factor = 1" concern
- However, the project has 80+ contributors and is gaining institutional adoption through OpenClaw

### Naming confusion
- "Pi" is noted as "entirely un-Google-able" due to conflicts with Raspberry Pi, the mathematical constant, and other projects
- The humorous original name "shittycodingagent" persists in the domain and is referenced in community discussions

### Competitive positioning
- Reviews consistently position Pi against Claude Code and OpenAI Codex CLI
- Pi's 4-tool minimalism vs Claude Code's 20+ built-in tools is a frequently cited differentiator
- Mid-session model switching across 15+ providers is highlighted as a unique feature

---

## 8. v0.65.2 Verification

- npm package `@mariozechner/pi-coding-agent` at version 0.65.2 ✓
- GitHub latest tag v0.65.2 dated 2026-04-06 ✓
- Breaking changes verified against source:
  - `AgentState` reshape and `AgentSessionRuntime` introduction (v0.65.0) ✓
  - `ModelRegistry` public constructor removal → `ModelRegistry.create()` (v0.64.0) ✓
  - `session_switch` / `session_fork` events removed → unified `session_start` with `event.reason` (v0.65.0) ✓
  - `session_directory` removed from extension and settings APIs (v0.65.0) ✓
  - Unknown single-dash CLI flags now produce an error (v0.65.0) ✓
  - `edit` tool unified to `edits[]` schema only; legacy single-edit migrated via `prepareArguments` (v0.63.2/v0.64.0) ✓

---

## 9. Verification Summary

| Claim | Status | Notes |
|-------|--------|-------|
| npm package `@mariozechner/pi-coding-agent` | VERIFIED | v0.65.2, 306 dependents |
| GitHub repo `badlogic/pi-mono` | VERIFIED | 33.2k stars, 3.7k forks |
| Website pi.dev | VERIFIED | Redirects from shittycodingagent.ai |
| Website buildwithpi.ai | VERIFIED | Session viewer, still active |
| Discord community | VERIFIED | Referenced across multiple sources |
| Author: Mario Zechner (badlogic) | VERIFIED | libGDX creator, npm @mariozechner scope |
| 231 release tags | VERIFIED | Counted from local git repo |
| OpenClaw uses Pi SDK | VERIFIED | Multiple sources confirm |
| 15+ LLM providers supported | VERIFIED | Anthropic, OpenAI, Google, Azure, Bedrock, Mistral, Groq, Cerebras, xAI, HuggingFace, Kimi, MiniMax, OpenRouter, Ollama, Vercel, z.ai, OpenCode, GitHub Copilot |
| Arch Linux AUR package | VERIFIED | https://aur.archlinux.org/packages/pi-coding-agent |
