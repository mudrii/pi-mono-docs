# Pi-Mono Documentation Review Plan and Execution Log

Date: 2026-05-11

Source repository: `/Users/mudrii/src/tui/pi-mono`  
Documentation repository: `/Users/mudrii/src/tui/pi-mono-docs`  
Latest fetched source commit: `3d9e14d7`  
Latest released tag: `v0.74.0` (`1eee081e`)  
DeepWiki source: `https://deepwiki.com/badlogic/pi-mono/1-overview`

## Goal

Review the current Pi monorepo source, released changelogs, local documentation, DeepWiki, and online package/release metadata. Update `pi-mono-docs` so current facts are correct for released versions only, with post-v0.74.0 `main` changes clearly marked as unreleased.

## Assumptions

- "Latest changes from main origins" means fetch/prune the configured source repo remote and fast-forward the local `main` branch.
- Released documentation should target `v0.74.0`, because npm and GitHub both report it as the latest release.
- Current `main` facts may be recorded only as unreleased notes.
- DeepWiki is an input source, not the final authority, because observed DeepWiki pages were indexed before v0.74.0.
- `graphify-out/GRAPH_REPORT.md` was required by the parent workspace instructions, but no `graphify-out` directory exists in this checkout, so graph-first review was not possible.

## Parallel Review Plan

1. Source synchronization -> verify: `git fetch --all --prune --tags` and `git pull --ff-only` in `pi-mono`.
2. Release boundary -> verify: compare local `main`, `v0.74.0`, package versions, changelogs, and npm dist-tags.
3. `packages/ai` review -> verify: package metadata, exports, provider IDs, auth envs, StreamOptions, model catalog, image APIs, and changelog from v0.72.1 through v0.74.0.
4. `packages/coding-agent` review -> verify: CLI flags, settings paths, session storage, tools, extensions, provider defaults, changelog, and released/unreleased split.
5. `packages/agent`, `packages/tui`, `packages/web-ui` review -> verify: public exports, package metadata, v0.73.x/v0.74.0 changes, and removed `mom`/pods status.
6. External documentation review -> verify: DeepWiki index age, GitHub release metadata, npm package metadata, old-scope vs new-scope package status.
7. Documentation edits -> verify: high-risk stale claims corrected in README, architecture, CLI, configuration, sessions, provider docs, version history, fact-check report, and package references.
8. Repository publication -> verify: commit only changed docs files, push to `origin/main`.

## Parallel Reviewer Findings

### ai

- `@earendil-works/pi-ai` is `0.74.0`.
- Released chat APIs are nine wire protocols; `openrouter-images` and `generateImages()` exist only on current `main` and are unreleased.
- Released v0.74.0 provider IDs include `cloudflare-ai-gateway`, `moonshotai`, `moonshotai-cn`, Xiaomi token-plan regional providers, and no Together AI.
- `StreamOptions` includes `transport: "sse" | "websocket" | "websocket-cached" | "auto"`, `timeoutMs`, `maxRetries`, and `onResponse(response: ProviderResponse, model)`.
- `google-gemini-cli` and `google-antigravity` were removed in v0.71.0.
- `supportsXhigh()` was removed in v0.72.0.

### coding-agent

- `@earendil-works/pi-coding-agent` is `0.74.0`.
- `pi [PROMPT]` does not by itself force print mode. Print mode is `--print`/`-p` or piped stdin.
- Current JSON/RPC modes are `--mode json` and `--mode rpc`, not `--json` or `--rpc`.
- `-ne`, `-ns`, and `-np` mean no extensions, no skills, and no prompt templates.
- Global settings are under `~/.pi/agent/settings.json`.
- Sessions are JSONL files under `~/.pi/agent/sessions/<encoded-cwd>/<timestamp>_<uuid>.jsonl`.
- v0.73.1 added JSONC `models.json` parsing and upstream `jiti` 2.7 extension loading.

### agent / tui / web-ui

- Active packages are `@earendil-works/pi-agent-core`, `@earendil-works/pi-tui`, and `@earendil-works/pi-web-ui` at `0.74.0`.
- `packages/mom` and `packages/pods` are not active current package directories.
- `AgentHarness` exports and related harness modules are current-main unreleased work.
- TUI v0.73.0/v0.73.1 released exact-match fuzzy ranking and inline image/hyperlink fixes.

### external docs

- DeepWiki is stale for current release facts because observed pages were indexed from April 2026 snapshots.
- GitHub release `v0.74.0` confirms the package/repository migration.
- npm confirms all active `@earendil-works/*` packages at `0.74.0`.
- Old active `@mariozechner/*` packages stop at `0.73.1`.

## Executed Documentation Changes

- Updated `README.md` to v0.74.0, 269 tags, new package scope, and DeepWiki staleness note.
- Updated `01-architecture-overview.md` for `@earendil-works/*`, `earendil-works/pi-mono`, and v0.74.0.
- Updated `cli-reference.md` for current print/json/rpc/session/resource flags.
- Updated `configuration.md` for `~/.pi/agent/settings.json`, `auth.json` key shape, and JSONC `models.json`.
- Updated `sessions.md` for actual JSONL session file layout and current resume/fork/session flags.
- Updated `providers.md` for current provider IDs, Xiaomi token-plan split, defaults, and unreleased Together note.
- Updated `02-pi-ai-llm-abstraction.md` for v0.74.0, current `StreamOptions`, nine released chat APIs, and provider ID corrections.
- Updated `03-pi-agent-core.md`, `04-pi-coding-agent.md`, `05-pi-tui-terminal-ui.md`, `06-application-packages.md`, and `07-extension-system.md` with current package/version/scope facts.
- Replaced `10-fact-check-report.md` with a current v0.74.0 fact-check report.
- Added v0.73.0, v0.73.1, v0.74.0, and unreleased-main sections to `09-version-history.md`.
- Updated current active package reference headers for `packages/agent.md`, `packages/ai.md`, `packages/tui.md`, and `packages/web-ui.md`.

## Verification Checklist

- Source repo fetched and fast-forwarded: completed.
- Latest release identified: `v0.74.0`, completed.
- npm dist-tags checked for active packages: completed.
- DeepWiki consulted and staleness recorded: completed.
- Parallel review completed across source scopes: completed.
- Docs edited with released/unreleased boundary: completed.
- Pending after this document: markdown grep checks, git status review, commit, push.
