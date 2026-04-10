# Pi-Mono Documentation Update to v0.66.1

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Update all pi-mono-docs from v0.65.2 to v0.66.1, fact-check all content against source code and online sources, and ensure documentation accuracy.

**Architecture:** The docs repo has 28 markdown files organized as 10 numbered core docs, supplementary references, and per-package references. We'll update each file with v0.66.0/v0.66.1 changes, correct stale statistics, and verify all claims against source CHANGELOGs and GitHub/npm data.

**Tech Stack:** Markdown documentation, git, GitHub CLI

---

## Key Facts (verified 2026-04-10)

- **Current version:** v0.66.1 (published 2026-04-08 on npm)
- **Total release tags:** 243 (up from 241)
- **GitHub stars:** 33.8k (up from 17.4k)
- **GitHub forks:** 3.8k (up from 1.8k)
- **All packages at:** 0.66.1 lockstep
- **v0.66.0 date:** 2026-04-08
- **v0.66.1 date:** 2026-04-08 (same day, later)

## Changes in v0.66.0 and v0.66.1

### coding-agent v0.66.0
- Earendil startup announcement with bundled inline image rendering
- Interactive Anthropic subscription auth warning
- Fixed `node:readline` import for Deno compat (#2885)
- Fixed auto-retry for stream failures (#2892)

### coding-agent v0.66.1
- Changed Earendil announcement to hidden `/dementedelves` slash command

### ai v0.66.0
- Fixed bare `readline` import for Deno compat (#2885)

### ai v0.66.1
- (no changes)

### tui (unreleased, on main)
- Fixed `Container.render()` stack overflow (#2651)

### agent v0.66.0/v0.66.1
- (no changes beyond version bump)

### mom/web-ui/pods v0.66.0/v0.66.1
- (no changes beyond version bump)

---

### Task 1: Update README.md (root index)

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Update version references from v0.65.2 to v0.66.1**

Change all occurrences of `v0.65.2` to `v0.66.1`, update the "Current release" line, the "Generated on" footer, the statistics section, and the "Releases covered" line.

Key changes:
- `Current release: v0.65.2 (2026-04-06)` → `Current release: v0.66.1 (2026-04-08)`
- `covers all released versions (v0.0.1 through v0.65.2)` → `covers all released versions (v0.0.1 through v0.66.1)`
- `Releases covered: 241 versions (v0.0.1 → v0.65.2)` → `Releases covered: 243 versions (v0.0.1 → v0.66.1)`
- `Community: 17.4k GitHub stars, 1.8k forks, 306 npm dependents` → `Community: 33.8k GitHub stars, 3.8k forks`
- `Packages (v0.65.2)` → `Packages (v0.66.1)`
- Footer: `Generated on 2026-04-09 from pi-mono source code at commit main (v0.65.2)` → `Generated on 2026-04-10 from pi-mono source code at commit main (v0.66.1)`

- [ ] **Step 2: Verify the update looks correct**

Read the file and confirm all version references are consistent.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: update README to v0.66.1 with corrected stats"
```

---

### Task 2: Update 01-architecture-overview.md

**Files:**
- Modify: `01-architecture-overview.md`

- [ ] **Step 1: Update version references**

Change all `0.65.2` to `0.66.1` throughout the document (package versions, lockstep versioning examples).

- [ ] **Step 2: Update total release count**

If the document mentions 241 releases, update to 243.

- [ ] **Step 3: Verify build system and dependency hierarchy are still accurate**

Cross-check against `pi-mono/package.json` and workspace config. The three-tier architecture (pi-ai → pi-agent-core → pi-coding-agent) hasn't changed.

- [ ] **Step 4: Commit**

```bash
git add 01-architecture-overview.md
git commit -m "docs: update architecture overview to v0.66.1"
```

---

### Task 3: Update 02-pi-ai-llm-abstraction.md

**Files:**
- Modify: `02-pi-ai-llm-abstraction.md`

- [ ] **Step 1: Update version references**

Change `0.65.2` to `0.66.1`.

- [ ] **Step 2: Add v0.66.0 changes**

In relevant sections, note:
- Fixed bare `readline` import to use `node:readline` prefix for Deno compatibility (#2885)

- [ ] **Step 3: Verify provider count and model catalog accuracy**

The doc claims 23+ providers. Verify by checking `packages/ai/src/types.ts` for the `Api` and `Provider` union types.

- [ ] **Step 4: Commit**

```bash
git add 02-pi-ai-llm-abstraction.md
git commit -m "docs: update pi-ai docs to v0.66.1"
```

---

### Task 4: Update 03-pi-agent-core.md

**Files:**
- Modify: `03-pi-agent-core.md`

- [ ] **Step 1: Update version references**

Change `0.65.2` to `0.66.1`.

- [ ] **Step 2: Verify AgentState API accuracy**

The v0.65.0 breaking changes reshaped AgentState significantly. Verify the documented API matches the current source in `packages/agent/src/`.

- [ ] **Step 3: Commit**

```bash
git add 03-pi-agent-core.md
git commit -m "docs: update pi-agent-core docs to v0.66.1"
```

---

### Task 5: Update 04-pi-coding-agent.md

**Files:**
- Modify: `04-pi-coding-agent.md`

- [ ] **Step 1: Update version references**

Change `0.65.2` to `0.66.1`.

- [ ] **Step 2: Add v0.66.0/v0.66.1 features**

Document:
- Earendil startup announcement (v0.66.0) → hidden `/dementedelves` command (v0.66.1)
- Interactive Anthropic subscription auth warning (v0.66.0)
- `PI_CODING_AGENT=true` env var set at startup (unreleased, on main)
- Deno compatibility fix for readline (#2885)
- Stream failure auto-retry improvement (#2892)

- [ ] **Step 3: Verify tool count and slash command list**

The doc claims 7 built-in tools and 18 slash commands. Verify against source.

- [ ] **Step 4: Commit**

```bash
git add 04-pi-coding-agent.md
git commit -m "docs: update pi-coding-agent docs to v0.66.1"
```

---

### Task 6: Update 05-pi-tui-terminal-ui.md

**Files:**
- Modify: `05-pi-tui-terminal-ui.md`

- [ ] **Step 1: Update version references**

Change `0.65.2` to `0.66.1`.

- [ ] **Step 2: Note Container.render() fix**

Document the stack overflow fix in Container.render() (#2651) which replaced `Array.push(...spread)` with a loop-based push to prevent `RangeError: Maximum call stack size exceeded`.

- [ ] **Step 3: Verify component count**

The doc claims 13 components. Verify against source in `packages/tui/src/`.

- [ ] **Step 4: Commit**

```bash
git add 05-pi-tui-terminal-ui.md
git commit -m "docs: update pi-tui docs to v0.66.1"
```

---

### Task 7: Update 06-application-packages.md

**Files:**
- Modify: `06-application-packages.md`

- [ ] **Step 1: Update version references**

Change `0.65.2` to `0.66.1`.

- [ ] **Step 2: Verify pi-web-ui, pi-mom, pi-pods content**

Cross-check against source READMEs and CHANGELOG files. No substantial changes in these packages for v0.66.0/v0.66.1.

- [ ] **Step 3: Commit**

```bash
git add 06-application-packages.md
git commit -m "docs: update application packages docs to v0.66.1"
```

---

### Task 8: Update 07-extension-system.md

**Files:**
- Modify: `07-extension-system.md`

- [ ] **Step 1: Update version references**

Change `0.65.2` to `0.66.1`.

- [ ] **Step 2: Verify extension API accuracy**

Cross-check `defineTool()` helper (added v0.65.0), `prepareArguments` hook (added v0.64.0), and `ctx.signal` (added v0.63.2) are documented.

- [ ] **Step 3: Commit**

```bash
git add 07-extension-system.md
git commit -m "docs: update extension system docs to v0.66.1"
```

---

### Task 9: Update 08-sessions-and-persistence.md

**Files:**
- Modify: `08-sessions-and-persistence.md`

- [ ] **Step 1: Update version references**

Change `0.65.2` to `0.66.1`.

- [ ] **Step 2: Verify AgentSessionRuntime documentation**

`AgentSessionRuntime` and `createAgentSessionRuntime()` were added in v0.65.0. Ensure these are properly documented including the factory pattern.

- [ ] **Step 3: Commit**

```bash
git add 08-sessions-and-persistence.md
git commit -m "docs: update sessions docs to v0.66.1"
```

---

### Task 10: Update 09-version-history.md

**Files:**
- Modify: `09-version-history.md`

- [ ] **Step 1: Add v0.66.0 and v0.66.1 entries**

Add to Era 7 section:

```markdown
### v0.66.0 (2026-04-08)
- **coding-agent**: Earendil startup announcement with bundled inline image rendering and linked blog post; interactive Anthropic subscription auth warning when subscription auth is active
- **ai**: Fixed bare `readline` import to use `node:readline` prefix for Deno compatibility (#2885)
- **coding-agent**: Fixed auto-retry to treat stream failures like `request ended without sending any chunks` as transient errors (#2892)

### v0.66.1 (2026-04-08)
- **coding-agent**: Changed Earendil announcement from automatic startup notice to hidden `/dementedelves` slash command
```

- [ ] **Step 2: Update overview statistics**

- Total tags: 241 → 243
- Latest version: v0.65.2 → v0.66.1
- Date range: through 2026-04-06 → through 2026-04-08
- Era 7 range: v0.61.0 – v0.65.2 → v0.61.0 – v0.66.1

- [ ] **Step 3: Update "Current release" reference in Era 6**

Era 6 header says `v0.60.0 (2026-03-18) -- Current release` which is wrong (was wrong even in v0.65.2 docs). Remove the "Current release" label.

- [ ] **Step 4: Commit**

```bash
git add 09-version-history.md
git commit -m "docs: add v0.66.0 and v0.66.1 to version history"
```

---

### Task 11: Update 10-fact-check-report.md

**Files:**
- Modify: `10-fact-check-report.md`

- [ ] **Step 1: Update verified statistics**

- GitHub stars: 17.4k → 33.8k
- GitHub forks: 1.8k → 3.8k
- npm version: 0.65.2 → 0.66.1
- Release count: 241 → 243

- [ ] **Step 2: Update verification date**

Change verification date to 2026-04-10.

- [ ] **Step 3: Verify pi.dev, shittycodingagent.ai, buildwithpi.ai domains**

Check that all three domains are still referenced in the source repo.

- [ ] **Step 4: Commit**

```bash
git add 10-fact-check-report.md
git commit -m "docs: update fact-check report for v0.66.1"
```

---

### Task 12: Update supplementary reference docs

**Files:**
- Modify: `changelog.md`
- Modify: `cli-reference.md`
- Modify: `configuration.md`
- Modify: `extensions.md`
- Modify: `providers.md`
- Modify: `sessions.md`
- Modify: `tools.md`

- [ ] **Step 1: Update changelog.md with v0.66.0 and v0.66.1 entries**

Add the new release entries from all package CHANGELOGs.

- [ ] **Step 2: Update version references in all supplementary docs**

Scan all files for `0.65.2` references and update to `0.66.1`.

- [ ] **Step 3: Verify CLI reference accuracy**

Cross-check CLI flags and commands against `packages/coding-agent/src/cli.ts`.

- [ ] **Step 4: Commit**

```bash
git add changelog.md cli-reference.md configuration.md extensions.md providers.md sessions.md tools.md
git commit -m "docs: update supplementary reference docs to v0.66.1"
```

---

### Task 13: Update package reference docs

**Files:**
- Modify: `packages/ai.md`
- Modify: `packages/agent.md`
- Modify: `packages/tui.md`
- Modify: `packages/web-ui.md`
- Modify: `packages/mom.md`
- Modify: `packages/pods.md`

- [ ] **Step 1: Update version references in all package docs**

Change `0.65.2` to `0.66.1` in all package reference files.

- [ ] **Step 2: Add relevant changes per package**

- ai.md: Deno readline fix
- tui.md: Container.render() stack overflow fix
- Others: version bump only

- [ ] **Step 3: Cross-check against source READMEs**

Verify package docs match the current source `packages/*/README.md` files.

- [ ] **Step 4: Commit**

```bash
git add packages/
git commit -m "docs: update package reference docs to v0.66.1"
```

---

### Task 14: Final verification and push

**Files:**
- All modified files

- [ ] **Step 1: Run a grep for any remaining v0.65.2 references**

```bash
grep -r "0\.65\.2" . --include="*.md" | grep -v ".git/"
```

All should be gone (except historical references in version history).

- [ ] **Step 2: Run a grep for stale statistics**

```bash
grep -rn "17.4k\|17,400\|1.8k\|1,800\|241 release\|241 version" . --include="*.md" | grep -v ".git/"
```

- [ ] **Step 3: Verify git status is clean**

```bash
git status
git log --oneline -15
```

- [ ] **Step 4: Push to GitHub**

```bash
git push origin main
```
