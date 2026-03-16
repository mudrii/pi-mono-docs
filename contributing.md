# Contributing

Pi is an open-source project built by a small core team and a community of contributors. This document explains how to contribute effectively.

---

## The One Rule

**You must understand your code.** If you can't explain what your changes do and how they interact with the rest of the system, your PR will be closed.

Using AI to write code is fine. You can gain understanding by interrogating an agent with access to the codebase until you grasp all edge cases and effects of your changes. What's not fine is submitting agent-generated code without that understanding.

If you use an agent, run it from the `pi-mono` root directory so it picks up `AGENTS.md` automatically.

---

## First-Time Contributors

Pi uses an approval gate for new contributors:

1. **Open an issue** describing what you want to change and why
2. Keep it concise — if it doesn't fit on one screen, it's too long
3. Write in your own voice, at least for the intro
4. A maintainer will comment `lgtm` if approved
5. Once approved, you can submit PRs

This exists because AI makes it trivial to generate plausible-looking but low-quality contributions. The issue step provides early filtering.

---

## Development Setup

### Prerequisites

- Node.js ≥ 20.6.0
- pnpm (package manager)
- Git

### Clone and Install

```bash
git clone https://github.com/badlogic/pi-mono
cd pi-mono
npm install
```

### Build Order

Packages have dependencies on each other. Build in this order:

```bash
cd packages/ai && npm run build
cd packages/tui && npm run build
cd packages/agent-core && npm run build
cd packages/coding-agent && npm run build
cd packages/web-ui && npm run build
cd packages/mom && npm run build
cd packages/pods && npm run build
```

Or build everything from the root:

```bash
npm run build  # Builds all packages in dependency order
```

### Run Checks

Before submitting a PR, all checks must pass:

```bash
npm run check  # TypeScript type checking + linting for all packages
./test.sh      # Integration tests
```

**Do not submit PRs where `npm run check` fails.**

### Development Mode

Run a package in watch mode:

```bash
cd packages/coding-agent
npm run dev  # Rebuild on file changes
```

**Do not run `npm run dev` or `npm test` unless you know what you're doing** — they may spawn processes or make network calls.

---

## Code Style

- **No `any` types** — use proper TypeScript types
- **Top-level imports only** — no inline `require()` or dynamic `import()` unless architecturally necessary
- **No inline comments** — code should be self-explanatory; if it's not, refactor it
- **No docstrings** on unchanged code — only add where genuinely needed for public APIs

---

## Monorepo Conventions

### Lockstep Versioning

All packages share the same version number. Do not bump version numbers in PRs — maintainers handle releases.

### Package Structure

```
packages/
├── ai/            # @mariozechner/pi-ai
├── tui/           # @mariozechner/pi-tui
├── agent-core/    # @mariozechner/pi-agent-core
├── coding-agent/  # @mariozechner/pi-coding-agent
├── web-ui/        # @mariozechner/pi-web-ui
├── mom/           # @mariozechner/pi-mom
└── pods/          # @mariozechner/pi-pods
```

### CHANGELOG

**Do not edit `CHANGELOG.md`.** Changelog entries are added by maintainers during releases.

If your PR adds a significant feature or fix, you may add a note under the `[Unreleased]` section — but this is optional and maintainers may rewrite it.

---

## Adding a New LLM Provider

Adding a provider to `packages/ai` requires:

1. Implement the streaming API for the new provider
2. Add the provider to the model registry in `models.json`
3. Add required tests (see `AGENTS.md` for the test requirements)
4. Document the setup method in your PR description

See the existing provider implementations in `packages/ai/src/providers/` for reference.

---

## Adding an Extension Example

Extension examples live in `packages/coding-agent/examples/extensions/`. Each example:

- Is a self-contained directory
- Has a `package.json` and TypeScript entry point
- Demonstrates one specific extension capability

See the [Extensions documentation](./extensions.md) for the extension API.

---

## Core Philosophy

Pi's core is **minimal by design**. When considering a feature:

- If it belongs in an **extension**, make it an extension
- If it requires a **new built-in tool**, think carefully — the 4 built-in tools are intentional
- If it adds **complexity without clear benefit to most users**, it will likely be rejected

PRs that bloat the core are declined regardless of code quality.

---

## Questions

- Open an issue on GitHub
- Ask on [Discord](https://discord.com/invite/nKXTsAcmbT)
- Use pi itself to explore the codebase — run `pi` from the `pi-mono` root

---

## Community

- **GitHub:** [github.com/badlogic/pi-mono](https://github.com/badlogic/pi-mono)
- **Discord:** [discord.com/invite/nKXTsAcmbT](https://discord.com/invite/nKXTsAcmbT)
- **npm:** [npmjs.com/~mariozechner](https://www.npmjs.com/~mariozechner)
- **X/Twitter:** [@badlogicgames](https://x.com/badlogicgames)
