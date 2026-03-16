# Pi Monorepo

> "There are many coding agents, but this one is mine."

**Current stable release: v0.58.3** (2026-03-15) — [GitHub](https://github.com/badlogic/pi-mono) · [Discord](https://discord.com/invite/3cU7Bz4UPx) · [pi.dev](https://pi.dev)

Pi is a minimal, extensible, terminal-based AI coding agent and supporting library ecosystem. Built by Mario Zechner (`badlogic`), it deliberately skips features like MCP, plan mode, sub-agents, and permission popups — keeping the core under 1,000 tokens of system prompt while allowing users to build exactly what they need via TypeScript extensions.

---

## Design Philosophy

Pi's design emerges from three frustrations with existing tools:

1. **Context opacity** — other agents inject context behind your back without surfacing it in the UI, making precise context engineering impossible
2. **Observability** — sub-agents operate as black boxes within black boxes
3. **Instability** — frequent system prompt changes break established workflows

**The result:** Primitives, not features. Only 4 built-in tools (`read`, `write`, `edit`, `bash`). Full filesystem and command access by default. Every feature lives in an extension.

> "If I don't need it, it won't be built." — Mario Zechner

---

## Packages

| Package | Version | Description |
|---------|---------|-------------|
| **[@mariozechner/pi-coding-agent](packages/coding-agent.md)** | 0.58.3 | Interactive coding agent CLI (`pi` binary) |
| **[@mariozechner/pi-ai](packages/ai.md)** | 0.58.3 | Unified multi-provider LLM API (23+ providers) |
| **[@mariozechner/pi-agent-core](packages/agent.md)** | 0.58.3 | Agent runtime with tool calling and state management |
| **[@mariozechner/pi-tui](packages/tui.md)** | 0.58.3 | Terminal UI library with differential rendering |
| **[@mariozechner/pi-web-ui](packages/web-ui.md)** | 0.58.3 | Web components for AI chat interfaces |
| **[@mariozechner/pi-mom](packages/mom.md)** | 0.58.3 | Slack bot delegating to pi coding agent |
| **[@mariozechner/pi-pods](packages/pods.md)** | 0.58.3 | CLI for managing vLLM deployments on GPU pods |

---

## Quick Start

```bash
npm install -g @mariozechner/pi-coding-agent
export ANTHROPIC_API_KEY=sk-ant-...
pi
```

Requires Node.js ≥ 20.6.0. See [Installation](installation.md) for all options.

---

## Key Features at a Glance

| Feature | Details |
|---------|---------|
| **Providers** | 23+ LLM providers, 100+ models (only tool-calling models included) |
| **System prompt** | Under 1,000 tokens (vs. 10,000+ in competing tools) |
| **Built-in tools** | 4: `read`, `write`, `edit`, `bash` |
| **Extensions** | TypeScript extensions with full API access |
| **Sessions** | Tree-structured JSONL with branching, forking, sharing |
| **Context files** | `AGENTS.md`, `SYSTEM.md`, skills, prompt templates |
| **Thinking** | `off`, `minimal`, `low`, `medium`, `high`, `xhigh` |
| **Terminal** | Differential rendering, Kitty protocol, IME, kill ring |
| **Platforms** | macOS, Linux, Windows (WSL/Git Bash), Android (Termux) |
| **License** | MIT |

---

## What Pi Intentionally Excludes

| Feature | Reason |
|---------|--------|
| MCP support | Standard MCP servers waste 7–9% of context with rarely-used tools |
| Sub-agents | Use bash to spawn agents for full visibility |
| Plan mode | AGENTS.md provides external observability |
| Permission popups | "YOLO by default" — full access |
| Built-in to-do | File-based tracking is clearer |
| Background bash | tmux provides better observability |

---

## Documentation

| Document | Description |
|----------|-------------|
| [Installation](installation.md) | Install methods, setup, first run |
| [CLI Reference](cli-reference.md) | All `pi` commands and flags |
| [Configuration](configuration.md) | Settings, environment variables, config files |
| [Architecture](architecture.md) | System design, data flow, package relationships |
| [LLM Providers](providers.md) | All supported providers and model selection |
| [Built-in Tools](tools.md) | `read`, `write`, `edit`, `bash` — the 4 core tools |
| [Extensions](extensions.md) | Extension API, lifecycle hooks, custom tools |
| [Sessions](sessions.md) | Session management, tree structure, sharing |
| [Context Engineering](context.md) | AGENTS.md, skills, prompt templates, compaction |
| [pi-ai Package](packages/ai.md) | Unified LLM API reference |
| [pi-agent-core Package](packages/agent.md) | Agent runtime API reference |
| [pi-tui Package](packages/tui.md) | Terminal UI library reference |
| [pi-web-ui Package](packages/web-ui.md) | Web components reference |
| [pi-mom Package](packages/mom.md) | Slack bot setup and usage |
| [pi-pods Package](packages/pods.md) | GPU pod management for vLLM |
| [Contributing](contributing.md) | Development setup and contribution guide |
| [Changelog](changelog.md) | Full release history |

---

## Community

- **GitHub:** 24.5k stars · 2.6k forks · 148 contributors · 173 releases
- **Discord:** https://discord.com/invite/3cU7Bz4UPx
- **Session sharing:** https://pi.dev
- **License:** MIT
