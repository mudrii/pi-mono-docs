# Pi-Mono Fact-Check Report

Date: 2026-05-11

This report verifies the current documentation against local source, release tags, npm metadata, GitHub release metadata, and DeepWiki. The latest released version is **v0.74.0**.

## Verified Release State

| Claim | Status | Evidence |
|-------|--------|----------|
| Latest released tag is `v0.74.0` | Verified | Local git tag list; GitHub release page for `v0.74.0` |
| Local `main` is newer than the release | Verified | Local `main` at `3d9e14d7`; `v0.74.0` tag at `1eee081e` |
| Active packages use lockstep `0.74.0` | Verified | `packages/{ai,agent,coding-agent,tui,web-ui}/package.json` |
| Active npm scope is `@earendil-works/*` | Verified | Source package metadata and npm `latest` dist-tags |
| Old `@mariozechner/*` active packages stop at `0.73.1` | Verified externally | npm package metadata |
| `pi-mom` and pods are not active workspace packages in v0.74.0 | Verified | No `packages/mom` or `packages/pods` package directories in current checkout |
| Released provider catalog excludes Together AI | Verified | `v0.74.0` source; Together appears only on current `main` after the release |

## npm Verification

The following packages are published at `0.74.0` with `latest` dist-tags:

- `@earendil-works/pi-coding-agent`
- `@earendil-works/pi-ai`
- `@earendil-works/pi-agent-core`
- `@earendil-works/pi-tui`
- `@earendil-works/pi-web-ui`

Legacy notes:

- `@mariozechner/pi-coding-agent`, `@mariozechner/pi-ai`, `@mariozechner/pi-agent-core`, `@mariozechner/pi-tui`, and `@mariozechner/pi-web-ui` remain available for older installs but should not be documented as current packages.
- `@mariozechner/pi-mom` and the legacy pods package are archived historical references. They are not active v0.74.0 workspace packages.

## GitHub Verification

- Canonical package repository metadata points at `https://github.com/earendil-works/pi-mono`.
- The v0.74.0 release states that repository links and package references moved to `earendil-works/pi-mono` and `@earendil-works/*`.
- `badlogic/pi-mono` URLs may redirect, but current documentation should use the `earendil-works/pi-mono` package metadata identity.

## DeepWiki Verification

DeepWiki remains useful for architectural orientation, but it is stale for current release facts.

- DeepWiki overview at `https://deepwiki.com/badlogic/pi-mono/1-overview` describes Pi as a monorepo whose primary artifact is the `pi` CLI.
- DeepWiki pages observed during this update were indexed from April 2026 snapshots such as `efc58f` and `156a90`.
- Those snapshots predate v0.73.0, v0.73.1, and v0.74.0, so they miss the `@earendil-works/*` scope migration and some package removals.
- DeepWiki pages that list `@mariozechner/*`, `pi-mom`, or pods as current package facts should be treated as stale.

## Released vs Unreleased Boundary

Released through v0.74.0:

- `@earendil-works/*` package scope.
- Xiaomi provider split into API billing `xiaomi` plus regional `xiaomi-token-plan-cn`, `xiaomi-token-plan-ams`, and `xiaomi-token-plan-sgp`.
- OAuth login flow metadata.
- JSONC-style `models.json` parsing.
- Upstream `jiti` 2.7 extension loading.
- Exact-match fuzzy ranking in TUI selector/autocomplete.

Unreleased on `main` after v0.74.0:

- Together AI provider and coding-agent login/default model wiring.
- OpenRouter image-generation API surface.
- `AgentHarness` and related session/resource/compaction modules in `pi-agent-core`.
- Fireworks session-affinity/caching fixes.
- TUI Markdown list indentation wrapping changes.

## Corrections Applied

- Updated current release references from `v0.72.1` to `v0.74.0`.
- Updated active package scope from `@mariozechner/*` to `@earendil-works/*`.
- Updated install and repository links to the current scope/repository.
- Corrected CLI mode flags: use `--print`/`-p`, `--mode json`, and `--mode rpc`; removed stale `--json`, `--rpc`, `--new-session`, and `--no-resume` claims.
- Corrected config/session storage paths to `~/.pi/agent/...`.
- Corrected provider IDs including `cloudflare-ai-gateway` and `moonshotai`.
- Added the v0.73.0, v0.73.1, and v0.74.0 release summaries.

## Residual Risk

Provider model names, prices, context windows, and endpoint behavior are time-sensitive. Current docs should treat local generated metadata as the project source of truth for v0.74.0, but external provider facts should be rechecked against official provider docs or models.dev before making pricing or capability claims outside the project source.
