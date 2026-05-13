# pi-mono Documentation

Comprehensive documentation for **pi** — a minimal, extensible, terminal-based AI coding agent and its supporting library ecosystem.

- **Release:** v0.74.0 (tag `v0.74.0`, commit `1eee081e`)
- **Author:** Mario Zechner ([@badlogic](https://github.com/badlogic))
- **Source:** [github.com/earendil-works/pi-mono](https://github.com/earendil-works/pi-mono)
- **CLI package:** [`@earendil-works/pi-coding-agent`](https://www.npmjs.com/package/@earendil-works/pi-coding-agent)
- **Website:** [pi.dev](https://pi.dev)
- **Discord:** [discord.com/invite/3cU7Bz4UPx](https://discord.com/invite/3cU7Bz4UPx)
- **License:** MIT

> All five packages share lockstep version `0.74.0`. The `@earendil-works/*` npm scope replaced the legacy `@mariozechner/*` scope. The user-config and session directory is `~/.pi/agent` (override with `PI_CODING_AGENT_DIR`).

---

## Quick navigation

| If you want to... | Read |
|---|---|
| Install pi and run a first session | [installation.md](installation.md) |
| Understand the five packages and how they layer | [01-architecture-overview.md](01-architecture-overview.md), [architecture.md](architecture.md) |
| Authenticate to a provider | [providers.md](providers.md) |
| Tune behavior via `settings.json` / `models.json` / `keybindings.json` | [configuration.md](configuration.md) |
| Drive pi from a shell, script, or editor | [cli-reference.md](cli-reference.md) |
| Inject project rules and skills | [context.md](context.md) |
| Know what the built-in tools do | [tools.md](tools.md) |
| Add tools, commands, providers, or UI | [07-extension-system.md](07-extension-system.md), [extensions.md](extensions.md) |
| Branch, fork, resume, or compact sessions | [08-sessions-and-persistence.md](08-sessions-and-persistence.md), [sessions.md](sessions.md) |
| Open an issue or PR | [contributing.md](contributing.md) |

---

## Architectural docs (numbered: deep references)

| # | Document | Scope |
|---|---|---|
| 01 | [Architecture Overview](01-architecture-overview.md) | Monorepo layout, three-tier dependency hierarchy, build system, lockstep versioning, release process |
| 02 | [pi-ai — LLM abstraction](02-pi-ai-llm-abstraction.md) | Provider IDs, API wire protocols, streaming events, ThinkingLevel mapping, OAuth, model catalog, cost tracking |
| 03 | [pi-agent-core](03-pi-agent-core.md) | `Agent` class, `agentLoop()`, parallel tool execution, lifecycle events, transport abstraction |
| 04 | [pi-coding-agent](04-pi-coding-agent.md) | CLI binary, modes (interactive/print/json/rpc/sdk), `AgentSession`, built-in tools, slash commands |
| 05 | [pi-tui — terminal UI](05-pi-tui-terminal-ui.md) | Components, Kitty keyboard protocol, differential rendering, overlays, autocomplete |
| 06 | [pi-web-ui — application packages](06-application-packages.md) | Browser web components for AI chat |
| 07 | [Extension system](07-extension-system.md) | Factory pattern, jiti loading, `ExtensionAPI`, events, tools, commands, providers, UI |
| 08 | [Sessions & persistence](08-sessions-and-persistence.md) | JSONL v3 tree format, branching, compaction, settings management, model resolution |
| 09 | [Version history](09-version-history.md) | Tag-by-tag changelog through v0.74.0 |
| 10 | [Fact-check report](10-fact-check-report.md) | Verification against local source at `v0.74.0` and stale external sources |

## Topic guides (focused references)

| Document | Scope |
|---|---|
| [installation.md](installation.md) | npm install, binary download, first-run auth, platform setup |
| [cli-reference.md](cli-reference.md) | All `pi` flags, modes, package subcommands, slash commands |
| [providers.md](providers.md) | All v0.74.0 provider IDs, env vars, `auth.json` keys, OAuth flows |
| [configuration.md](configuration.md) | `settings.json`, `auth.json`, `models.json`, env vars |
| [architecture.md](architecture.md) | Data-flow diagrams for an interactive session |
| [tools.md](tools.md) | `read`, `write`, `edit`, `bash`, `grep`, `find`, `ls`; bash spawn hook |
| [extensions.md](extensions.md) | Focused walkthrough of writing an extension |
| [sessions.md](sessions.md) | Day-to-day session management |
| [context.md](context.md) | `AGENTS.md` / `CLAUDE.md`, `SYSTEM.md`, skills, prompt templates |
| [contributing.md](contributing.md) | Approval gate, quality bar, dev setup, parallel-agent git rules |
| [changelog.md](changelog.md) | Synthesized release history (auto-distilled) |

## Package READMEs

| Package | Document |
|---|---|
| `@earendil-works/pi-ai` | [packages/ai.md](packages/ai.md) |
| `@earendil-works/pi-agent-core` | [packages/agent.md](packages/agent.md) |
| `@earendil-works/pi-tui` | [packages/tui.md](packages/tui.md) |
| `@earendil-works/pi-web-ui` | [packages/web-ui.md](packages/web-ui.md) |

---

## Packages at a glance (v0.74.0)

| Package | Tier | Purpose |
|---|---|---|
| **[@earendil-works/pi-ai](02-pi-ai-llm-abstraction.md)** | 1 — Foundation | Unified multi-provider LLM streaming API. No internal deps. |
| **[@earendil-works/pi-tui](05-pi-tui-terminal-ui.md)** | 1 — Foundation | Terminal UI components with differential rendering. No internal deps. |
| **[@earendil-works/pi-agent-core](03-pi-agent-core.md)** | 2 — Infrastructure | Agent runtime, tool loop, event streaming. Depends on pi-ai. |
| **[@earendil-works/pi-coding-agent](04-pi-coding-agent.md)** | 3 — Application | `pi` CLI binary, sessions, extensions, skills. Depends on all of the above. |
| **[@earendil-works/pi-web-ui](06-application-packages.md)** | 3 — Application | Browser web components (mini-lit + Tailwind v4). Depends on pi-ai and pi-tui. |

Build order: `pi-tui → pi-ai → pi-agent-core → pi-coding-agent → pi-web-ui`.

---

## Key facts (v0.74.0)

| Item | Value |
|---|---|
| **Node.js** | ≥ 20.6.0 |
| **CLI binary** | `pi` |
| **Default active tools** | `read`, `bash`, `edit`, `write` |
| **Available tools** | adds `grep`, `find`, `ls` (read-only) |
| **System prompt** | under 1,000 tokens |
| **Storage root** | `~/.pi/agent/` (override `PI_CODING_AGENT_DIR`) |
| **Sessions** | `~/.pi/agent/sessions/<encoded-cwd>/<ts>_<uuid>.jsonl` (JSONL v3 tree) |
| **Compaction defaults** | `reserveTokens: 16384`, `keepRecentTokens: 20000` |
| **Thinking levels** | `off`, `minimal`, `low`, `medium`, `high`, `xhigh` |
| **Modes** | interactive (default), `-p`/`--print`, `--mode json`, `--mode rpc`, SDK |
| **Subscription logins** | Claude Pro/Max, ChatGPT Plus/Pro (Codex), GitHub Copilot |
| **TypeBox** | `typebox` 1.x (migrated from `@sinclair/typebox` 0.34 in v0.69.0; legacy alias kept) |
| **Platforms** | macOS, Linux, Windows (Git Bash/WSL), Android (Termux) |

---

## Source-of-truth pointers

When a docs page conflicts with the source, the source wins. Authoritative locations:

- `packages/coding-agent/docs/*.md` — first-party user docs (installed under the `docs/` folder of the published package)
- `packages/coding-agent/src/cli/args.ts` — CLI flags and modes (canonical help output)
- `packages/coding-agent/src/core/model-resolver.ts` — `defaultModelPerProvider` map
- `packages/ai/src/env-api-keys.ts` — env-var to provider mapping
- `packages/ai/src/providers/register-builtins.ts` — lazy-loaded built-in providers
- `AGENTS.md` and `CONTRIBUTING.md` at repo root — contributor rules

---

Updated against `/Users/mudrii/src/tui/pi-mono` at tag `v0.74.0` (commit `1eee081e29c1323c40b98db11d0a62b919831881`).
