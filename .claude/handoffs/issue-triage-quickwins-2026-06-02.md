---
name: issue-triage-quickwins-2026-06-02
date: 2026-06-02 17:55
project: codegraph
branch: main
summary: Triaged all 72 open issues against the changelog, then shipped the already-fixed closes + 10 quick-win fixes (PR #654) + status --json CI fields (PR #655); both merged.
---

# Handoff: open-issue triage → quick-win fixes + status --json (#329)

## Resume here — read this first
**Current state:** All of this session's work is **merged to `main`** (tip `7b62356`), working tree clean, nothing in progress. 18 issues closed, 2 PRs merged, 2 superseded PRs closed. The triage surfaced a backlog of real bugs/features that are documented below but **not started**.
**Immediate next step:** Pick the next work item — either start a Tier-1 correctness bug or review the 4 in-flight contributor PRs. Highest-leverage correctness bug with the clearest entry point is **#629** (Svelte re-export barrels → false "0 callers").

> Suggested next message: "Let's fix #629 — start by making the default-export branch of `findExportedSymbol` (src/resolution/import-resolver.ts:~1250) also match node kind `component` so `export { default as X } from './X.svelte'` resolves, then check the bare `./`-index and package-subpath barrel forms. Reproduce on a SvelteKit repo first."

## Goal
Work the open-issue backlog: close what's already fixed, ship the cheap wins, and tee up the real bugs. This session's slice is **done and merged**; the goal now is to keep going on the remaining triaged items (Tier-1 correctness bugs, in-flight PR reviews, or the multi-root feature cluster).

## Key findings
- **Full triage of 72 open issues** was done via 6 parallel `general-purpose` agents, each anchoring issues to the reporter's version vs `CHANGELOG.md` (per CLAUDE.md's version-anchoring rule). Result buckets are in the conversation; the still-open real work is under "Open threads" below.
- **#329 design decision:** shipped `version`, `indexPath`, `lastIndexed` (ISO-8601 string, or null); **dropped `agentCount`** — it had no clear consumer, two conflicting meanings (configured integrations vs. live daemon sessions), and didn't fit the issue's own CI use case. Field names match the issue (not eddieran's `codegraphVersion`/ms+ISO scheme).
- **#329 impl:** `QueryBuilder.getLastIndexedAt()` = `SELECT MAX(indexed_at) FROM files` (epoch-ms; `indexed_at` is `Date.now()` at index time, schema.sql:64). Exposed as `CodeGraph.getLastIndexedAt(): number|null` (src/index.ts) and surfaced in both JSON branches of the `status` action (src/bin/codegraph.ts).
- **Quick-win fix locations** (all on `main` now): `.codegraph/.gitignore` → `*`+`!.gitignore` (src/directory.ts, BOTH write sites ~86 and ~248); MCP `resources/list`/`prompts/list` empty replies (src/mcp/session.ts + src/mcp/proxy.ts); extension map adds (src/extraction/grammars.ts); `getUnresolvedReferencesByFiles` chunking (src/db/queries.ts:~1588); `git ls-files -z` (src/extraction/index.ts, BOTH calls in `collectGitFiles`); Go generic-receiver regex (src/extraction/languages/go.ts:~60); anonymous-body visit (src/extraction/tree-sitter.ts:~633); impact `contains`-edge exclusion (src/graph/traversal.ts:~525).
- **Competing-PR handling (confirmed pattern):** for #329's two PRs (#333, #480) we shipped a fresh combined impl, credited BOTH via `Co-Authored-By:` trailers, and closed each with a specific thanks. Saved to memory `feedback_pr_improve_on_contributor_branch`.

## Gotchas
- **#583 is only HALF done.** Shipped RC1 (generic-receiver regex). RC2 — receiver methods declared in a *different file* from their struct lose the struct→method `contains` edge (src/extraction/tree-sitter.ts:~788 restricts owner lookup to same file) — is **still open**, needs a resolution-phase package-wide owner join. Don't tell anyone #583 is closed.
- **`gh issue close --comment` is unreliable** — comment first (`gh issue comment --body-file`), then close, then verify the comment is the last activity. Used this throughout.
- **`main` is REVIEW_REQUIRED** → merge with `gh pr merge <N> --squash --admin --delete-branch`. Co-authored-by needs numeric IDs: `gh api users/<login> --jq .id` → `<id>+<login>@users.noreply.github.com` (12122J=199902626, eddieran=8403607).
- **No `git add -p` in this env.** To split mixed working-tree changes into 2 branches: commit all to `tmp/snapshot`, branch each off `main`, `git checkout tmp/snapshot -- <owned files>`, and for shared files start from main + re-apply hunks via Edit. CHANGELOG `[Unreleased]` auto-merges when one PR adds `### New Features` and the other adds to `### Fixes`.
- Spawn-based tests (mcp-initialize, status-json) exec `dist/bin/codegraph.js` — **rebuild dist** before running them.
- `codegraph status` takes a **positional** path arg, not `--path` (that flag silently no-ops for status).

## How to test & validate
- `npm run build && npm test` → 59 files, ~1122 passed / 2 skipped on `main`. (Spawn tests need the build first.)
- `npx vitest run __tests__/<file>.test.ts` for a single file.
- Smoke test the real binary: `CODEGRAPH_NO_DAEMON=1 node dist/bin/codegraph.js status --json` (initialized) / `... status /tmp/empty --json` (uninitialized).
- For new language/framework coverage bugs (#629, #645, #608, #578…): follow the CLAUDE.md validation methodology — deterministic probe on the built `dist/` + agent A/B; verify "the flow EXISTS end-to-end in the graph" and node count is stable (no explosion).

## Repo state
- branch `main`, last commit `7b62356 feat(cli): add version, indexPath, lastIndexed to status --json (#329)`
- previous: `ddb1a8f fix: issue-triage quick wins (#654)`
- uncommitted: clean

## Open threads / TODO
Tier-1 correctness bugs (silently-wrong / zero-recall — highest value):
- [ ] **#645** C++ method calls via singletons/factories/chained getters → wrong class (needs C++ return-type extraction; L)
- [ ] **#629** Svelte default re-export barrels → false 0 callers (start: match kind `component` in `findExportedSymbol`, import-resolver.ts:~1250; M)
- [ ] **#608** PHP `Cls::for($x)->method()` static-factory chains drop the edge (M)
- [ ] **#578** Python `module.func()` after `from pkg import module` → zero recall (import-resolver.ts `resolveViaImport`; M)
- [ ] **#583-RC2** Go cross-file receiver methods lose `contains` edge (tree-sitter.ts:~788; M) → unblocks **#584** (Go structural `implements` edges)
- [ ] **#527** symlink read-escape (validatePathWithinRoot has no realpath check; `isPathWithinRootReal` exists but unused in read paths; security; S/M)

Platform: [ ] #237 (C# primary ctors — stale tree-sitter-wasms C# grammar, current is 0.23.5) · [ ] #448 (WAL on SMB/network drives) · [ ] #208 (Windows NTFS scan hang, mis-titled) · [ ] #576 (Windows daemon lingers after Ctrl+C)

In-flight contributor PRs to review/merge: [ ] #597 (#515 C macros) · [ ] #301 (#300 Drupal patterns) · [ ] #306 (#305 docs Contributing) · [ ] #57 (#55 MQL5; cookbook PR #97 was closed)

Framework coverage: [ ] #307 (Hono sub-routers) · [ ] #491/#490/#489 (Spring @ConditionalOnProperty / Feign→controller / MyBatis SqlSessionTemplate) · [ ] #634 (TS string-literal type args)

Feature cluster (one design pass, not piecemeal): [ ] multi-root / scan-boundary — #542, #518, #511, #499, #452, #304, #281, #141, #514, #636

Needs-info / verify-version-then-maybe-close: [ ] #641 (Codex MCP — PATH/timeout) · [ ] #501 (Antigravity 2.0 — UI enable?) · [ ] #493 (sync delete — confirm reporter <0.9.5) · [ ] #535 (opencode Windows %APPDATA% — needs real Windows VM) · [ ] #613 (10GB-DB MCP timeout — large-repo query profiling)

## Recent transcript (last 7 turns, oldest → newest)
### Turn 1 — "Scan all open issues; which are already fixed / quick wins / real?"
- **Claude:** `gh issue list` (72 open) + read CHANGELOG; built a version→fix map; fanned out 6 `Agent(general-purpose)` batches to triage 12 issues each, each verifying against `src/`.
- **Outcome:** Bucketed triage report: 7 already-fixed, ~11 quick wins, ~20 real bugs, ~20 features, tracking/dups.

### Turn 2 — "take care of the quick wins" (+ idea: append `&& codegraph install` to README curl/irm)
- **Claude:** Verified install.sh/install.ps1 do NOT refresh current-shell PATH → chaining would fail "command not found". Implemented 10 quick wins + tests + CHANGELOG; `npm test` green (1122).
- **Outcome:** All quick wins done locally; README reworked to an explicit `codegraph install` step "in a new terminal".

### Turn 3 — "handle these [#86 #316 #339 #342 #507 #551 #565], then look at 329 and the PR"
- **Claude:** Pinned fix-versions via `git tag --contains`; `gh issue comment --body-file` then `gh issue close` then verified, for all 7. Found **two** competing PRs for #329 (#333 by 12122J, #480 by eddieran).
- **Outcome:** 7 already-fixed issues closed. Surfaced the #329 PR conflict + the `agentCount` ambiguity.

### Turn 4 — "what is this feature?" → "why would anyone need agentCount" → "yes"
- **Claude:** Explained #329 (machine-readable `status` for CI). Argued `agentCount` has no real consumer → recommended dropping it. Implemented version/indexPath/lastIndexed + `getLastIndexedAt()` + `__tests__/status-json.test.ts`; green.
- **Outcome:** #329 implemented, agentCount dropped, validated end-to-end.

### Turn 5 — "Option A" (two separate PRs)
- **Claude:** Split the mixed working tree via `tmp/snapshot` + selective `git checkout` + Edit; built two clean branches (`fix/issue-triage-quick-wins`, `feat/status-json-ci-fields`), each green independently.
- **Outcome:** Two committed branches; main untouched.

### Turn 6 — "Yes" (push + open PRs)
- **Claude:** `git push` both; `gh pr create` → **#654** (closes 10 issues, refs #583) and **#655** (closes #329). Both MERGEABLE.
- **Outcome:** Two PRs open and verified.

### Turn 7 — "do it" (merge + close superseded)
- **Claude:** `gh pr merge 654 --squash --admin`; merged `main` into #655 branch (CHANGELOG auto-merged), pushed, `gh pr merge 655 --squash --admin` with co-author trailers. Closed #333/#480 with specific thanks; left a #583 RC2 note. Verified all auto-closes.
- **Outcome:** Everything merged; 18 issues closed; #583 open with note. Updated memory `feedback_pr_improve_on_contributor_branch`.
