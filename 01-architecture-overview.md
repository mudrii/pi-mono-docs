# Architecture Overview

## Project Overview

Pi is an open-source, LLM-powered coding agent system created by Mario Zechner (@badlogic). The project is branded as "the shitty coding agent" (website: shittycodingagent.ai) and is published under the `@mariozechner/*` namespace on npm. The primary domain is pi.dev, donated by exe.dev.

Pi is structured as a monorepo (`badlogic/pi-mono`) containing seven packages that together provide:

- A unified multi-provider LLM API
- A general-purpose agent runtime
- A terminal-based coding agent CLI
- A terminal UI framework
- Web components for AI chat interfaces
- A Slack bot integration
- A GPU pod management CLI for vLLM deployments

The coding agent (`pi`) is the flagship application. It ships with four core tools (read, write, edit, bash) and is designed around a minimal-core philosophy: instead of building in sub-agents and plan modes, pi exposes an extension system (TypeScript extensions, skills, prompt templates, themes, and pi packages) so users can adapt it to their workflows without forking.

Pi runs in four modes: interactive TUI, print/JSON output, RPC for process integration, and an SDK for embedding in other applications.

Current version: 0.60.0. License: MIT.

---

## Monorepo Structure

The repository uses npm workspaces. All seven packages live under `packages/`:

| Package | npm Name | Directory | Description |
|---------|----------|-----------|-------------|
| **pi-tui** | `@mariozechner/pi-tui` | `packages/tui` | Terminal UI library with differential rendering, synchronized output (CSI 2026), component system, inline images, and autocomplete |
| **pi-ai** | `@mariozechner/pi-ai` | `packages/ai` | Unified multi-provider LLM API supporting 20+ providers (OpenAI, Anthropic, Google, Mistral, Bedrock, etc.) with automatic model discovery, token/cost tracking, and cross-provider handoffs |
| **pi-agent-core** | `@mariozechner/pi-agent-core` | `packages/agent` | Stateful agent runtime with tool execution, event streaming, transport abstraction, and state management |
| **pi-coding-agent** | `@mariozechner/pi-coding-agent` | `packages/coding-agent` | Interactive coding agent CLI with session management, extension system, skills, prompt templates, and themes |
| **pi-mom** | `@mariozechner/pi-mom` | `packages/mom` | "Master Of Mischief" -- a self-managing Slack bot that delegates messages to the pi coding agent, runs in Docker sandboxes, and builds its own tools autonomously |
| **pi-web-ui** | `@mariozechner/pi-web-ui` | `packages/web-ui` | Web components (built with mini-lit and Tailwind CSS v4) for AI chat interfaces with attachments, artifacts, IndexedDB storage, and CORS proxy support |
| **pi-pods** | `@mariozechner/pi` | `packages/pods` | CLI for managing vLLM deployments on GPU pods (DataCrunch, RunPod, Vast.ai, etc.) with automatic setup and model configuration |

The workspace configuration in the root `package.json` also includes several example extension directories as workspace members:

```
packages/web-ui/example
packages/coding-agent/examples/extensions/with-deps
packages/coding-agent/examples/extensions/custom-provider-anthropic
packages/coding-agent/examples/extensions/custom-provider-gitlab-duo
packages/coding-agent/examples/extensions/custom-provider-qwen-cli
```

---

## Three-Tier Dependency Hierarchy

The packages form a three-tier dependency tree. Each tier builds on the one below it.

```
Tier 3 - Applications
  pi-coding-agent ─────┬── pi-agent-core ── pi-ai
                       ├── pi-ai
                       └── pi-tui
  pi-mom ──────────────┬── pi-coding-agent
                       ├── pi-agent-core
                       └── pi-ai
  pi-web-ui ───────────┬── pi-ai
                       └── pi-tui
  pi-pods ─────────────┬── pi-agent-core

Tier 2 - Infrastructure
  pi-agent-core ───────── pi-ai
  pi-tui ─────────────── (no internal deps)

Tier 1 - Foundation
  pi-ai ──────────────── (no internal deps)
  pi-tui ─────────────── (no internal deps)
```

### Tier 1: Foundation

- **pi-ai** -- The LLM abstraction layer. Has no dependencies on other pi packages. Provides a unified streaming API across providers, model discovery, tool definition via TypeBox/Zod schemas, token tracking, and context serialization. Providers are lazily loaded via subpath exports (e.g., `@mariozechner/pi-ai/anthropic`).
- **pi-tui** -- The terminal UI framework. Also has no internal dependencies. Provides differential rendering, component abstractions (Text, Editor, Markdown, SelectList, Image, overlays, etc.), and input handling.

### Tier 2: Infrastructure

- **pi-agent-core** -- Depends on pi-ai. Adds stateful agent semantics on top of the raw LLM API: message management (`AgentMessage`), turn-based tool execution loops, event streaming (agent_start, turn_start, message_update, etc.), and context transformation pipelines.

### Tier 3: Applications

- **pi-coding-agent** -- Depends on pi-ai, pi-agent-core, and pi-tui. The main user-facing CLI application.
- **pi-mom** -- Depends on pi-ai, pi-agent-core, and pi-coding-agent. Wraps the coding agent for Slack integration.
- **pi-web-ui** -- Depends on pi-ai and pi-tui. Provides browser-side chat components.
- **pi-pods** -- Depends on pi-agent-core. GPU pod management tooling.

### Rationale

This layering enforces separation of concerns. pi-ai can be used standalone for any LLM integration. pi-agent-core can power agents that have nothing to do with coding. The coding agent, Slack bot, and web UI are all consumers of these lower layers. This also means changes to foundation packages require careful consideration since they affect everything above.

---

## Build System

### npm Workspaces

The monorepo uses native npm workspaces (Node.js >= 20.0.0 required). Dependencies are hoisted to the root `node_modules`, and inter-package references use `^version` semver ranges kept in sync by `scripts/sync-versions.js`.

### TypeScript

The project uses TypeScript with a shared base configuration in `tsconfig.base.json`:

- **Target**: ES2022
- **Module system**: Node16 (ESM with `.js` extensions)
- **Strict mode**: enabled
- **Declarations**: `.d.ts` and declaration maps generated for all packages

The root `tsconfig.json` extends the base config with `noEmit: true` and path aliases mapping `@mariozechner/*` package names to their source directories. This enables IDE navigation across packages without building first. Each package has its own `tsconfig.build.json` for compilation.

The project uses `tsgo` (from `@typescript/native-preview`) as the build compiler for most packages, and standard `tsc` for pi-web-ui (which needs browser-compatible output).

### Biome

Code formatting and linting use Biome (v2.3.5), configured in the root `biome.json`:

- **Indent**: tabs, width 3
- **Line width**: 120
- **Linting**: recommended rules with some relaxations (`noExplicitAny: off`, `noNonNullAssertion: off`)
- **Scope**: only `packages/*/src/**/*.ts` and `packages/*/test/**/*.ts` files

### Lockstep Versioning

All seven packages share the same version number (currently 0.60.0). The `scripts/sync-versions.js` script enforces this by:

1. Reading all package versions and verifying they match
2. Updating all inter-package `dependencies` and `devDependencies` to `^<current-version>`

Version bumps are performed atomically across all packages via `npm version <type> -ws`.

### Build Order

The build script in the root `package.json` enforces a sequential build order that respects the dependency graph:

```
pi-tui -> pi-ai -> pi-agent-core -> pi-coding-agent -> pi-mom -> pi-web-ui -> pi-pods
```

Note: `npm run check` requires `npm run build` to have been run first because the web-ui package uses `tsc` which needs compiled `.d.ts` files from dependencies.

---

## Development Workflow

### Core Commands

```bash
npm install          # Install all dependencies
npm run build        # Build all packages in dependency order
npm run check        # Lint (biome), format, type check (tsgo --noEmit), browser smoke test
./test.sh            # Run tests (skips LLM-dependent tests without API keys)
./pi-test.sh         # Run pi from sources (must be run from repo root)
```

### Per-Package Testing

Tests use vitest (pi-ai, pi-agent-core, pi-coding-agent) or Node.js built-in test runner (pi-tui). Tests are run from the package root:

```bash
npx tsx ../../node_modules/vitest/dist/cli.js --run test/specific.test.ts
```

### Development Mode

```bash
npm run dev          # Watch mode for all packages (via concurrently)
```

### AGENTS.md Rules

The `AGENTS.md` file defines rules for both human and AI contributors:

- Never run `npm run dev`, `npm run build`, or `npm test` (use specific test files)
- Always run `npm run check` after code changes and fix all errors before committing
- No `any` types unless absolutely necessary
- No inline imports (no `await import()`, no dynamic imports for types)
- Never hardcode keybindings; use configurable binding objects
- Never remove code to fix type errors from outdated deps; upgrade instead

---

## Release Process

Releases are managed by `scripts/release.mjs`. All packages are released together at the same version.

### Version Semantics

The project does not use major version bumps. The convention is:

- **patch** (`npm run release:patch`): Bug fixes and new features
- **minor** (`npm run release:minor`): API breaking changes

### Release Steps

The release script performs these steps automatically:

1. Verify clean working directory (no uncommitted changes)
2. Bump version across all packages (`npm run version:<type>`, which calls `npm version <type> -ws` then `sync-versions.js`)
3. Update all `CHANGELOG.md` files: replace `## [Unreleased]` with `## [<version>] - <date>`
4. Commit with message `Release v<version>` and create git tag `v<version>`
5. Publish all packages to npm with `--access public`
6. Add new `## [Unreleased]` sections to all changelogs
7. Commit changelog updates
8. Push to `origin main` and push the version tag

### Changelog Convention

Each package maintains its own `CHANGELOG.md` with sections under `## [Unreleased]`:

- **Breaking Changes** -- API changes requiring migration
- **Added** -- New features
- **Changed** -- Changes to existing functionality
- **Fixed** -- Bug fixes
- **Removed** -- Removed features

Contributors do not edit changelogs; maintainers add entries. Released version sections are immutable.

---

## Contributing Guidelines

Contributions follow an approval gate process defined in `CONTRIBUTING.md`:

1. **Open an issue first** describing the proposed change and its rationale
2. Keep it concise (one screen max), written in your own voice
3. A maintainer comments `lgtm` to approve
4. Only then may you submit a PR

### The One Rule

"You must understand your code." Using AI to write code is allowed, but submitting agent-generated changes without understanding how they interact with the system will get the PR closed.

### Before Submitting

```bash
npm run check   # Must pass with no errors
./test.sh       # Must pass
```

### PR Workflow

The project does not use contributor-opened PRs in the traditional sense. The maintainer workflow (from AGENTS.md) is: analyze the PR without pulling, then if approved, create a feature branch, pull, rebase on main, apply adjustments, merge into main, and push.

### OSS Weekend Mode

The project has an "OSS weekend" mechanism (`scripts/oss-weekend.mjs`) that auto-closes issues from non-maintainers and PRs from approved non-maintainers during designated periods.

---

## Key Design Philosophy

### Minimal Core

Pi's core is intentionally small. From `CONTRIBUTING.md`: "If your feature doesn't belong in the core, it should be an extension. PRs that bloat the core will likely be rejected."

The coding agent ships with only four tools (read, write, edit, bash) and deliberately omits features like sub-agents and plan mode. Users add capabilities through:

- **Extensions** -- TypeScript modules loaded at runtime via a custom jiti fork
- **Skills** -- Slash commands that inject prompts or trigger actions
- **Prompt Templates** -- Reusable system prompt configurations
- **Themes** -- Visual customization of the TUI
- **Pi Packages** -- Shareable bundles of extensions/skills/templates/themes distributed via npm or git

### Provider Agnosticism

pi-ai supports 20+ LLM providers through a unified streaming API. Only models with tool-calling support are included, since tool use is essential for agentic workflows. Providers are lazily loaded to avoid bundling unused SDKs.

### Layered SDK

The three-tier architecture means each layer is independently usable:

- Use pi-ai alone for multi-provider LLM access
- Use pi-agent-core to build custom agents without the coding-agent opinions
- Use pi-coding-agent as a CLI or embed it via SDK (as OpenClaw does)

---

## Scripts Directory

The `scripts/` directory contains build and release tooling:

| Script | Purpose |
|--------|---------|
| `release.mjs` | Automated release: version bump, changelog update, commit, tag, publish, push |
| `sync-versions.js` | Enforces lockstep versioning across all inter-package dependencies |
| `check-browser-smoke.mjs` | Verifies pi-ai can be imported in a browser-like environment |
| `oss-weekend.mjs` | Toggles OSS weekend mode (auto-closes external issues/PRs) |
| `build-binaries.sh` | Builds standalone binaries (using bun compile) |
| `cost.ts` | Utility for cost calculations |
| `session-transcripts.ts` | Utility for session transcript handling |
| `browser-smoke-entry.ts` | Entry point for the browser smoke test |
