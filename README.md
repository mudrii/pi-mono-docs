# Pi Monorepo Documentation

Comprehensive documentation for **Pi** — a minimal, extensible, terminal-based AI coding agent and supporting library ecosystem.

- **Current release:** v0.74.0 (2026-05-07)
- **Author:** Mario Zechner ([@badlogic](https://github.com/badlogic))
- **Package:** [`@earendil-works/pi-coding-agent`](https://www.npmjs.com/package/@earendil-works/pi-coding-agent) on npm
- **Source:** [github.com/earendil-works/pi-mono](https://github.com/earendil-works/pi-mono)
- **Website:** [pi.dev](https://pi.dev) / [shittycodingagent.ai](https://shittycodingagent.ai)
- **Discord:** [discord.com/invite/3cU7Bz4UPx](https://discord.com/invite/3cU7Bz4UPx)

> This documentation covers all released versions (v0.0.1 through v0.74.0). All current-release content is fact-checked against local source at `3d9e14d7`, tag `v0.74.0`, npm metadata, GitHub release metadata, and DeepWiki where it remains current.

---

## Design Philosophy

Pi's core is deliberately minimal — under 1,000 tokens of system prompt, only 4 built-in tools (`read`, `write`, `edit`, `bash`), full filesystem access by default. Every feature lives in an extension. This stands in contrast to agents with 10,000+ token system prompts and dozens of built-in capabilities.

> "If I don't need it, it won't be built." — Mario Zechner

---

## Core Documentation

| # | Document | Description |
|---|----------|-------------|
| 01 | [Architecture Overview](01-architecture-overview.md) | Monorepo structure, three-tier dependency hierarchy, build system, lockstep versioning, release process, contributing guidelines |
| 02 | [Pi-AI: LLM Abstraction](02-pi-ai-llm-abstraction.md) | Released provider IDs, API identifiers, streaming entry points, ThinkingLevel mappings, OAuth, model catalog, cost tracking, TypeBox tool validation |
| 03 | [Pi-Agent-Core](03-pi-agent-core.md) | Agent class API, loop execution model, tool pipeline (parallel by default), 10 event types, transport abstraction, low-level API |
| 04 | [Pi-Coding-Agent](04-pi-coding-agent.md) | CLI modes (interactive/print/RPC/SDK), AgentSession, 7 built-in tools, JSONL sessions, compaction, settings, 18 slash commands |
| 05 | [Pi-TUI: Terminal UI](05-pi-tui-terminal-ui.md) | 13 components, Kitty keyboard protocol, differential rendering, autocomplete, fuzzy matching, overlay system |
| 06 | [Application Packages](06-application-packages.md) | pi-web-ui (ChatPanel, browser, web components) |
| 07 | [Extension System](07-extension-system.md) | Factory pattern, jiti loading, ExtensionAPI, hooks, UI integration, custom providers, example extensions |
| 08 | [Sessions & Persistence](08-sessions-and-persistence.md) | JSONL tree structure, branching, compaction, settings management, model resolution, migration |
| 09 | [Version History](09-version-history.md) | 269 tags through v0.74.0, breaking changes, feature milestones, unreleased-main boundary |
| 10 | [Fact-Check Report](10-fact-check-report.md) | Verification against local source, npm, GitHub releases, and stale DeepWiki pages |

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

---

## Packages (v0.74.0)

| Package | Description |
|---------|-------------|
| **[@earendil-works/pi-coding-agent](04-pi-coding-agent.md)** | Interactive coding agent CLI (`pi` binary) |
| **[@earendil-works/pi-ai](02-pi-ai-llm-abstraction.md)** | Unified multi-provider LLM API |
| **[@earendil-works/pi-agent-core](03-pi-agent-core.md)** | Agent runtime with tool calling and state management |
| **[@earendil-works/pi-tui](05-pi-tui-terminal-ui.md)** | Terminal UI library with differential rendering |
| **[@earendil-works/pi-web-ui](06-application-packages.md)** | Web components for AI chat interfaces |

---

## Key Features

| Feature | Details |
|---------|---------|
| **Providers** | 29 released provider IDs in v0.74.0, generated tool-calling model metadata |
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
- **Source files reviewed:** 170+ TypeScript files across 5 packages
- **Providers documented:** 29 released provider IDs with API identifiers
- **Releases covered:** 269 tags (v0.0.1 -> v0.74.0)
- **Online verifications:** npm package metadata, GitHub release metadata, DeepWiki index state

---

Updated on 2026-05-11 from pi-mono source code at commit `3d9e14d7` with latest release tag `v0.74.0`.
Cross-referenced with [DeepWiki](https://deepwiki.com/badlogic/pi-mono/1-overview), npm package metadata, and GitHub releases. DeepWiki remains useful for architecture orientation but is stale for package scope and releases after April 2026.
