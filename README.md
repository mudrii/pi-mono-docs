# Pi Monorepo Documentation

Comprehensive documentation for **Pi** — a minimal, extensible, terminal-based AI coding agent and supporting library ecosystem.

- **Current release:** v0.60.0 (2026-03-19)
- **Author:** Mario Zechner ([@badlogic](https://github.com/badlogic))
- **Package:** [`@mariozechner/pi-coding-agent`](https://www.npmjs.com/package/@mariozechner/pi-coding-agent) on npm
- **Source:** [github.com/badlogic/pi-mono](https://github.com/badlogic/pi-mono)
- **Website:** [pi.dev](https://pi.dev) / [shittycodingagent.ai](https://shittycodingagent.ai)
- **Discord:** [discord.com/invite/3cU7Bz4UPx](https://discord.com/invite/3cU7Bz4UPx)

> This documentation covers all released versions (v0.0.1 through v0.60.0). All content is fact-checked against source code, deepwiki documentation, and online sources.

---

## Design Philosophy

Pi's core is deliberately minimal — under 1,000 tokens of system prompt, only 4 built-in tools (`read`, `write`, `edit`, `bash`), full filesystem access by default. Every feature lives in an extension. This stands in contrast to agents with 10,000+ token system prompts and dozens of built-in capabilities.

> "If I don't need it, it won't be built." — Mario Zechner

---

## Core Documentation

| # | Document | Description |
|---|----------|-------------|
| 01 | [Architecture Overview](01-architecture-overview.md) | Monorepo structure, three-tier dependency hierarchy, build system, lockstep versioning, release process, contributing guidelines |
| 02 | [Pi-AI: LLM Abstraction](02-pi-ai-llm-abstraction.md) | 23 providers, API identifiers, streaming entry points, ThinkingLevel mappings, OAuth, model catalog, cost tracking, TypeBox tool validation |
| 03 | [Pi-Agent-Core](03-pi-agent-core.md) | Agent class API, loop execution model, tool pipeline (parallel by default), 10 event types, transport abstraction, low-level API |
| 04 | [Pi-Coding-Agent](04-pi-coding-agent.md) | CLI modes (interactive/print/RPC/SDK), AgentSession, 7 built-in tools, JSONL sessions, compaction, settings, 18 slash commands |
| 05 | [Pi-TUI: Terminal UI](05-pi-tui-terminal-ui.md) | 13 components, Kitty keyboard protocol, differential rendering, autocomplete, fuzzy matching, overlay system |
| 06 | [Application Packages](06-application-packages.md) | pi-web-ui (ChatPanel, browser), pi-mom (Slack bot, sandbox), pi-pods (GPU management, vLLM) |
| 07 | [Extension System](07-extension-system.md) | Factory pattern, jiti loading, ExtensionAPI, hooks, UI integration, custom providers, example extensions |
| 08 | [Sessions & Persistence](08-sessions-and-persistence.md) | JSONL tree structure, branching, compaction, settings management, model resolution, migration |
| 09 | [Version History](09-version-history.md) | All 231 releases in 6 eras, breaking changes, feature milestones |
| 10 | [Fact-Check Report](10-fact-check-report.md) | Online verification: npm (306 dependents), GitHub (17.4k stars), domains, Discord, author |

## Supplementary Reference

Previously generated reference docs covering v0.58.4:

| Document | Description |
|----------|-------------|
| [Installation](installation.md) | Install methods, setup, first run |
| [CLI Reference](cli-reference.md) | All `pi` commands and flags |
| [Configuration](configuration.md) | Settings, environment variables, config files |
| [Architecture](architecture.md) | System design, data flow, package relationships |
| [LLM Providers](providers.md) | Provider list and model selection |
| [Built-in Tools](tools.md) | `read`, `write`, `edit`, `bash` tool reference |
| [Extensions](extensions.md) | Extension API, lifecycle hooks |
| [Sessions](sessions.md) | Session tree structure, sharing |
| [Context Engineering](context.md) | AGENTS.md, skills, prompt templates |
| [Contributing](contributing.md) | Development setup and contribution guide |
| [Changelog](changelog.md) | Release history summary |

### Package References

| Document | Description |
|----------|-------------|
| [pi-ai](packages/ai.md) | Unified LLM API reference |
| [pi-agent-core](packages/agent.md) | Agent runtime API reference |
| [pi-tui](packages/tui.md) | Terminal UI library reference |
| [pi-web-ui](packages/web-ui.md) | Web components reference |
| [pi-mom](packages/mom.md) | Slack bot setup and usage |
| [pi-pods](packages/pods.md) | GPU pod management |

---

## Packages (v0.60.0)

| Package | Description |
|---------|-------------|
| **[@mariozechner/pi-coding-agent](04-pi-coding-agent.md)** | Interactive coding agent CLI (`pi` binary) |
| **[@mariozechner/pi-ai](02-pi-ai-llm-abstraction.md)** | Unified multi-provider LLM API (23+ providers) |
| **[@mariozechner/pi-agent-core](03-pi-agent-core.md)** | Agent runtime with tool calling and state management |
| **[@mariozechner/pi-tui](05-pi-tui-terminal-ui.md)** | Terminal UI library with differential rendering |
| **[@mariozechner/pi-web-ui](06-application-packages.md)** | Web components for AI chat interfaces |
| **[@mariozechner/pi-mom](06-application-packages.md)** | Slack bot delegating to pi coding agent |
| **[@mariozechner/pi-pods](06-application-packages.md)** | CLI for managing vLLM deployments on GPU pods |

---

## Key Features

| Feature | Details |
|---------|---------|
| **Providers** | 23+ LLM providers, 100+ models (tool-calling only) |
| **System prompt** | Under 1,000 tokens |
| **Built-in tools** | 7: `read`, `write`, `edit`, `bash`, `grep`, `find`, `ls` |
| **Extensions** | TypeScript extensions with full API access via jiti |
| **Sessions** | Tree-structured JSONL with branching and compaction |
| **Thinking** | `off`, `minimal`, `low`, `medium`, `high`, `xhigh` |
| **Terminal** | Differential rendering, Kitty protocol, IME support |
| **Platforms** | macOS, Linux, Windows (WSL/Git Bash), Android (Termux) |
| **License** | MIT |

---

## Statistics

- **Total documentation:** ~20,000+ lines across 28 files
- **Source files reviewed:** 170+ TypeScript files across 7 packages
- **Providers documented:** 23+ with API identifiers
- **Releases covered:** 231 versions (v0.0.1 → v0.60.0)
- **Online verifications:** npm, GitHub, 3 domains, Discord, author confirmed
- **Community:** 17.4k GitHub stars, 1.8k forks, 306 npm dependents

---

Generated on 2026-03-20 from pi-mono source code at commit `main` (v0.60.0).
Cross-referenced with [deepwiki.com/badlogic/pi-mono](https://deepwiki.com/badlogic/pi-mono).
