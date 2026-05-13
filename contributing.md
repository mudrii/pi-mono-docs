# Contributing

Pi is open source. This document distills the authoritative `CONTRIBUTING.md` and `AGENTS.md` at the repo root. When this page conflicts with those files, those files win.

---

## The One Rule

**You must understand your code.** If you can't explain what your changes do and how they interact with the rest of the system, your PR will be closed.

Using AI to write code is fine. You can gain understanding by interrogating an agent (including pi itself) with access to the codebase until you grasp all edge cases and effects. What's not fine is submitting agent-generated code without that understanding.

If you use an agent, run it from the `pi-mono` root so it picks up `AGENTS.md` automatically.

---

## First-Time Contributors — Approval Gate

Pi uses an approval gate for new contributors:

1. **Open an issue** describing what you want to change and why.
2. Keep it concise. If it doesn't fit on one screen, it's too long.
3. Write in your own voice, at least for the intro.
4. A maintainer will comment **`lgtmi`** if the issue is approved. Approved issues unlock submission of *that issue's* PR.
5. A maintainer will comment **`lgtm`** if both the issue and the resulting PR are approved.
6. **Weekend issues are not reviewed.** Reviewers are on weekdays only.
7. Bypassing the gate or pinging maintainers to escalate may result in being blocked from the project.

The gate exists because AI makes it trivial to generate plausible-looking but low-quality contributions. Filtering at the issue step saves everyone's time.

---

## Development Setup

### Prerequisites

- Node.js ≥ 20.6.0
- npm (the monorepo uses npm workspaces, not pnpm)
- Git

### Clone and Install

```bash
git clone https://github.com/earendil-works/pi-mono
cd pi-mono
npm install
```

### Build

From the repo root, the build script enforces dependency order:

```bash
npm run build   # tui → ai → agent → coding-agent → web-ui
```

To build a single package, build its dependencies first. The five active packages are under `packages/`:

```
packages/tui          → @earendil-works/pi-tui
packages/ai           → @earendil-works/pi-ai
packages/agent        → @earendil-works/pi-agent-core
packages/coding-agent → @earendil-works/pi-coding-agent
packages/web-ui       → @earendil-works/pi-web-ui
```

(The legacy `pi-mom` and `pi-pods` packages were removed on 2026-04-30, between v0.70.6 and v0.71.0; see commit `0ed0d434`.)

### Pre-PR Checks

Both must pass:

```bash
npm run check   # TypeScript + lint across all packages
./test.sh       # Integration tests
```

Do not submit PRs where `npm run check` fails.

### Development Mode

Run a package in watch mode:

```bash
cd packages/coding-agent
npm run dev
```

Do not run `npm run dev` or `npm test` blindly — they may spawn processes or make network calls.

### Running Pi from Source

Use the repo-root helper script:

```bash
./pi-test.sh [pi flags]
```

This runs the locally built `pi` against your live source so changes are picked up immediately after `npm run build`.

---

## Code Style

From `AGENTS.md`:

- **No `any` types** — use proper TypeScript types.
- **Top-level imports only** — no inline `require()` or dynamic `import()` unless architecturally necessary.
- **No inline comments** — code should be self-explanatory; refactor if it isn't.
- **No docstrings** on unchanged code — only add where genuinely needed for public APIs.
- **No emojis** in commits, code, or PR descriptions.
- **Never hardcode keybindings** — go through the keybinding registry.
- **Never modify `packages/ai/src/models.generated.ts`** directly — it's regenerated from upstream sources.

---

## Monorepo Conventions

### Lockstep Versioning

All five packages share the same version. Maintainers handle releases; do not bump versions in PRs.

### CHANGELOG

**Do not edit `CHANGELOG.md`.** Changelog entries are added by maintainers during release. If your PR is significant you may note it in the PR description; maintainers may rewrite or drop the note.

---

## Parallel-Agent Git Rules

When running multiple agents in parallel branches/worktrees of the same repo, follow these rules (verbatim from `AGENTS.md`):

- **ONLY commit files YOU changed in THIS session.** Run `git status` before staging; don't sweep up another agent's work.
- **ALWAYS use `git add <specific-file-paths>`** listing only files you modified. Never `git add -A` or `git add .` — they sweep up changes from other agents.
- **ALWAYS include `fixes #<number>` or `closes #<number>`** in commit messages when there is a related issue or PR.

Forbidden destructive ops (do not run, do not suggest):

- `git reset --hard`
- `git checkout .`
- `git clean -fd`
- `git stash` / `git stash pop`
- `git commit --no-verify` / `--no-gpg-sign`
- Force pushes to `main`/`master`

If a hook fails, fix the cause and create a new commit. Do **not** `--amend` (the failed commit didn't happen, so `--amend` modifies the *previous* commit and risks rewriting unrelated work).

---

## Adding a New LLM Provider

A provider addition to `packages/ai` requires:

1. Implement the streaming protocol in `packages/ai/src/providers/`.
2. Register the provider in `packages/ai/src/providers/register-builtins.ts` (lazy registration map) and add its env-var binding to `packages/ai/src/env-api-keys.ts` if it uses one.
3. Add the appropriate default model entry to `packages/coding-agent/src/core/model-resolver.ts`.
4. Add provider tests; see `AGENTS.md` for the test charter.
5. Update first-party docs at `packages/coding-agent/docs/providers.md`.
6. Document the setup method in your PR description.

The model registry (`packages/ai/src/models.generated.ts`) is regenerated from upstream sources — do not hand-edit.

---

## Adding an Extension Example

Extension examples live under `packages/coding-agent/examples/extensions/`. Each example:

- Is a self-contained directory with its own `package.json`.
- Has a TypeScript entry point exported via the `pi.extensions` field in `package.json`, or a top-level `index.ts`.
- Demonstrates one specific extension capability.

See [Extensions](extensions.md) for the extension API and the bundled examples for patterns.

---

## Core Philosophy

Pi's core is **minimal by design**. When considering a feature, ask:

- Can it be an **extension**? Make it an extension. Extension-able features are not accepted into core.
- Does it require a **new built-in tool**? Think carefully — the four default tools are intentional.
- Does it add **complexity without clear benefit to most users**? It will likely be declined.

PRs that bloat the core are declined regardless of code quality.

---

## Community

- **Issues / PRs:** [github.com/earendil-works/pi-mono](https://github.com/earendil-works/pi-mono)
- **Discord:** [discord.com/invite/3cU7Bz4UPx](https://discord.com/invite/3cU7Bz4UPx)
- **Website:** [pi.dev](https://pi.dev)
- **X/Twitter:** [@badlogicgames](https://x.com/badlogicgames)
