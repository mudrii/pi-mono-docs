# Pi-Mono Fact-Check Report

Date: 2026-05-13
Anchor: v0.74.0 (commit `1eee081e29c1323c40b98db11d0a62b919831881`)
Scope: Final verification of freshly-rewritten docs in `/Users/mudrii/src/tui/pi-mono-docs/` against source at `/Users/mudrii/src/tui/pi-mono` (tag v0.74.0), live npm metadata, and DeepWiki staleness boundaries.

This report records source-grep verification for high-risk claims and flags residual inconsistencies. The earlier 2026-05-11 version of this file has been replaced.

---

## Verified Release State

| Claim | Status | Evidence |
|-------|--------|----------|
| Latest released tag is `v0.74.0` | Verified | `git describe --tags` -> `v0.74.0`; `git log -1` -> `1eee081e Release v0.74.0` |
| Active packages use lockstep `0.74.0` | Verified | `packages/{ai,agent,coding-agent,tui,web-ui}/package.json` all at 0.74.0 |
| Active npm scope is `@earendil-works/*` | Verified | npm `latest` dist-tag = 0.74.0 for all 5 packages |
| Legacy `@mariozechner/*` frozen at `0.73.1` | Verified | `npm view @mariozechner/pi-coding-agent version` = 0.73.1; same for pi-ai |
| `pi-mom` and `pi-pods` not active in v0.74.0 | Verified | No `packages/mom` or `packages/pods` directories; removed at commit `0ed0d434` on 2026-04-30 |
| Released provider catalog excludes Together AI | Verified | `grep -r together packages/ai/src` returns nothing at v0.74.0; Together appears only on later `main` |

---

## npm Verification (executed live)

| Package | `latest` dist-tag | Expected | Status |
|---------|-------------------|----------|--------|
| `@earendil-works/pi-coding-agent` | 0.74.0 | 0.74.0 | OK |
| `@earendil-works/pi-ai` | 0.74.0 | 0.74.0 | OK |
| `@earendil-works/pi-agent-core` | 0.74.0 | 0.74.0 | OK |
| `@earendil-works/pi-tui` | 0.74.0 | 0.74.0 | OK |
| `@earendil-works/pi-web-ui` | 0.74.0 | 0.74.0 | OK |
| `@mariozechner/pi-coding-agent` | 0.73.1 | 0.73.1 (frozen) | OK |
| `@mariozechner/pi-ai` | 0.73.1 | 0.73.1 (frozen) | OK |

Scope migration to `@earendil-works/*` confirmed by npm registry.

---

## GitHub Verification

| Claim | Status | Notes |
|-------|--------|-------|
| Repository now at `earendil-works/pi-mono` | Trusted | Per release-process doc and scope migration; not re-fetched live |
| Old `badlogic/pi-mono` references in legacy DeepWiki crawls | Expected stale | DeepWiki indexed commit `156a9052` (2026-04-30) predates rename |
| Release `v0.74.0` tagged 2026-05-xx | Verified locally | `git tag -l v0.74.0`; commit message `Release v0.74.0` |

---

## DeepWiki Staleness Boundaries

DeepWiki index at commit `156a9052` (2026-04-30) is pre-v0.74.0 and pre-scope-rename. Any DeepWiki claim involving:

- `pi-mom` or `pi-pods` packages -> stale; removed at `0ed0d434` (2026-04-30).
- `@mariozechner/*` scope -> stale for active packages; legacy packages still resolve but frozen at 0.73.1.
- Repo URL `badlogic/pi-mono` -> stale; now `earendil-works/pi-mono`.
- File line numbers in any package source -> high-risk stale; v0.74.0 added compaction reshuffling and TypeBox 1.x migration deltas.
- Provider count, default model IDs, and tool list -> verify against source, not DeepWiki.

The docs correctly mention the DeepWiki staleness anchor (`156a9052`) where relevant.

---

## Source-Grep Verifications (v0.74.0)

| Claim | Source | Status |
|-------|--------|--------|
| 7 built-in tools: read, bash, edit, write, grep, find, ls | `packages/coding-agent/src/core/tools/index.ts:83-84` | Verified |
| Default Anthropic model = `claude-opus-4-7` | `packages/coding-agent/src/core/model-resolver.ts:14-23` | Verified |
| Default OpenAI model = `gpt-5.4` | same file | Verified |
| Default OpenAI Codex model = `gpt-5.5` | same file | Verified |
| Default Google / Vertex model = `gemini-3.1-pro-preview` | same file | Verified |
| Bedrock default = `us.anthropic.claude-opus-4-6-v1` (claude-opus-4-6, intentional) | same file | Verified, NOT a typo |
| Storage root = `~/.pi/agent/` with override `PI_CODING_AGENT_DIR` | `packages/coding-agent/src/config.ts:426-459` | Verified |
| Thinking levels: `off, minimal, low, medium, high, xhigh` | `packages/coding-agent/src/cli/args.ts` VALID_THINKING_LEVELS | Verified |
| Mode = `text | json | rpc` | `packages/coding-agent/src/cli/args.ts` Mode type | Verified |
| CLI flags catalog matches `cli-reference.md` | `packages/coding-agent/src/cli/args.ts` | Verified, full enumeration |
| Compaction defaults `reserveTokens=16384`, `keepRecentTokens=20000` | source constants in agent-core | Verified |
| 9 wire-protocol APIs registered | `packages/ai/src/providers/register-builtins.ts:342-396` (9 `registerApiProvider` calls) | Verified |

---

## Resolved Ambiguity: `enabledTools` vs `tools`

Earlier agent flagged a possible `enabledTools` settings key in user-facing examples. Source grep:

```
grep -rn "enabledTools" /Users/mudrii/src/tui/pi-mono/packages/coding-agent/src/
-> NO MATCHES
```

Settings field for model cycling is `enabledModels: string[]` (for Ctrl+P). Tool allowlisting at session start uses `AgentSessionConfig.initialActiveToolNames` and `allowedToolNames`. The CLI surface uses `--tools` (allowlist). No settings key named `enabledTools` exists. If any doc still mentions `enabledTools`, it is wrong; corrected docs use `--tools` (CLI) and `initialActiveToolNames` (programmatic).

---

## Released vs Unreleased Boundary

| Item | State | Source |
|------|-------|--------|
| Together AI provider | Unreleased | Mentioned only as "current `main` after v0.74.0" in `providers.md:104-106`; no `together` strings in v0.74.0 source |
| `KnownProvider` union | Released | `packages/ai/src/types.ts:19-50` -> 31 IDs |
| Removed `pi-mom`, `pi-pods` | Removed before v0.74.0 | Commit `0ed0d434` on 2026-04-30; pre-v0.71.0 |
| TypeBox 1.x migration | Released in v0.69.0 | Per changelog |
| JSONL v3 session tree | Released | Active format in v0.74.0 |

`providers.md` correctly separates released vs unreleased — Together AI flagged as not released.

---

## Corrections Applied Per Agent (Verified in Place)

| Agent / Doc | Correction | Verified |
|-------------|------------|----------|
| 01 / Architecture | npm scope migrated to `@earendil-works/*` | Yes |
| 02 / pi-ai | Lazy provider loading via subpath exports | Yes |
| 03 / pi-agent-core | Transport abstraction described | Yes |
| 04 / pi-coding-agent | CLI flag list matches `args.ts`; mode enum text/json/rpc | Yes |
| 05 / pi-tui | CSI 2026 synchronized output | Yes |
| 06 / Application packages | mom/pods removal dated 2026-04-30 at `0ed0d434` | Yes |
| 07 / Extensions | `pi.*` vs `ctx.*` API split documented | Yes |
| 08 / Sessions | Default models table matches `model-resolver.ts` | Yes |
| 09 / Version history | v0.74.0 anchored as latest | Yes |
| `tools.md` | 7-tool count (was earlier flagged as 4) | Yes |
| `configuration.md` | Compaction keys correct | Yes |

Bedrock default `claude-opus-4-6` retained intentionally (user-confirmed; source explicitly has 4-6 while other Anthropic provider defaults are 4-7). Not a typo.

---

## Issues Flagged for Follow-Up

### A. MAJOR: Provider count contradiction across docs (and both wrong)

- `01-architecture-overview.md:30` says **"29+ released providers"**.
- `01-architecture-overview.md:259` says **"29 released LLM provider IDs in v0.74.0"**.
- `providers.md:3` says **"29 released LLM provider IDs in v0.74.0"**.
- `02-pi-ai-llm-abstraction.md:12` says **"30 model providers across 9 wire protocols"**.
- `02-pi-ai-llm-abstraction.md:76` says **"Union of 30 built-in provider IDs"**.

Source truth at v0.74.0: `packages/ai/src/types.ts:19-50` declares **31 KnownProvider IDs**.

`providers.md` table (lines 11-40) enumerates **30 rows**. Diff against source: `fireworks` (declared in `types.ts` and registered with env var `FIREWORKS_API_KEY` at `env-api-keys.ts:119`) is **missing** from the `providers.md` table.

Wire-protocol count "9" in `02` is correct (9 `registerApiProvider` calls at `register-builtins.ts:342-396`).

Recommended fix: update all four numeric claims to `31` and add a `fireworks` row to the `providers.md` provider table (env var `FIREWORKS_API_KEY`).

### B. MINOR: docs/providers.md fireworks omission

Same root cause as A. The provider exists in code and env-key map but has no row in the user-facing provider table.

### C. INFO: Internal inconsistency in `02` vs `01` over phrasing

Even after fixing the count, `01` says "29+ released providers" (a soft estimate) while `02` claims a hard count of "Union of N built-in provider IDs". Pick one style — recommend hard count from `KnownProvider`.

### D. INFO: `providers.md:138-145` Thinking levels table lists `xhigh` notes "GPT-5.2+, Claude Opus 4.6" — verify whether v0.74.0's default Anthropic `claude-opus-4-7` and Bedrock `claude-opus-4-6` both support `xhigh`. Not blocking.

---

## Residual Risk

- Time-sensitive provider env-var names verified against `env-api-keys.ts:101-129` only spot-checked (Anthropic, OpenAI, Fireworks). A full diff between `providers.md` "Auth Method" column and `env-api-keys.ts` was not performed.
- Pricing / model-metadata generation pipeline not re-verified end-to-end; docs claim "automatic model discovery" — source has `models.ts` registry + getProviders, consistent with claim.
- `web-ui` end-to-end behavior (IndexedDB, CORS proxy) not source-grepped this pass; relies on prior agent's reading.
- `xiaomi-token-plan-*` regional providers documented per v0.73.0 changelog; not behavior-tested.
- Bedrock 4-6 default vs Anthropic 4-7 default is intentional per user; rationale (older AWS marketplace listing) is not stated in docs. Consider a single line in `providers.md` to pre-empt confusion.

---

## Recommended Further Verification

1. Run `npm view @earendil-works/pi-ai versions --json | tail -5` to confirm no post-0.74.0 release exists yet.
2. `grep -rn "enabledTools" packages/` -> confirm zero matches (already done; zero).
3. Diff `providers.md` Auth Method column against `packages/ai/src/env-api-keys.ts:101-129` field-by-field.
4. Add `fireworks` row to `providers.md` table (currently the only KnownProvider ID without a documented row).
5. Update provider count to 31 in: `01-architecture-overview.md` (two places), `02-pi-ai-llm-abstraction.md` (two places), `providers.md` (one place).
6. After fixes, re-run a final `grep -rn "29\|30 model providers\|30 built-in" pi-mono-docs/` to ensure no stale counts remain.

---

## Summary

- Source-grep verifications passed: 18.
- npm dist-tag verifications passed: 7 (5 active + 2 legacy frozen).
- Issues flagged: 4 (1 major, 1 minor, 2 info).
- Major contradiction: provider count claimed as 29 or 30 across multiple docs; actual count in source is 31, and `providers.md` table is missing the `fireworks` row.
- Bedrock `claude-opus-4-6` default is intentional per source and user; not a typo.
- DeepWiki staleness boundary correctly anchored at commit `156a9052`.
