---
name: impact-coverage-per-language-2026-06-03
date: 2026-06-03 21:30
project: codegraph
branch: main
summary: Fixed the engine's impact/affected tool (file-dependent coverage 62.5%→95.8%) for TypeScript; now replicating per-language.
---

# Handoff: Make impact/`affected` work across every language

## Resume here — read this first
**Current state:** TypeScript is DONE and validated (full suite green, 1134 passed). Three fixes landed (uncommitted on `main`): the `affected`/`getFileDependents` query fix, in-body type-annotation extraction, and import/re-export linking. The latter took file-dependent coverage on this repo from **62.5% → 95.8%** (residual 5 files are all correctly-zero: worker scripts + see-through barrels + a CLI entry). I had just asked (via AskUserQuestion) which language to do next — Python / Java / Go / C# — and the user rejected the question and ran `/handoff save` instead.
**Immediate next step:** Pick the next language and replicate the methodology: measure coverage on a real repo → audit the 0-dependent files → fix extraction/resolution → validate node-stability + tests. Don't re-ask with a tool; just start (the user declined the menu).

> Suggested next message: "Let's do Python next. Index a real Python repo, measure file-dependent coverage the same way, audit the 0-dependent files, and tell me what the real gaps are before we fix anything."

## Goal
The engine's impact tool (symbol-level `getImpactRadius` and file-level `getFileDependents`/`affected`) must capture **all real cross-file dependencies** for every supported language — recall-first ("never miss the affected feature"). "Coverage" = % of source files with ≥1 cross-file dependent. 100% is the WRONG target (entry points / workers / see-through barrels genuinely have 0); the real bar is **no real dependency missed**, validated by auditing every 0-dependent file.

## Key findings (TS work — all landed, uncommitted)
- **`affected` was returning 0 for every file.** Root cause: `imports` edges in this graph are **same-file** (`file → its own local import nodes`); `getFileDependents` followed imports-only → 0 cross-file dependents for 72/72 files. Fix: `src/db/queries.ts` new `getDependentFilePaths`/`getDependencyFilePaths` (one indexed JOIN, all edge kinds except `contains`); `src/graph/queries.ts` `getFileDependents`/`getFileDependencies` now delegate. Un-breaks `findCircularDependencies` too.
- **In-body type annotations were dropped.** `visitFunctionBody` (`src/extraction/tree-sitter.ts` ~line 2143) extracted calls but never type annotations, so `const items: Foo[] = []` inside any function/method/object body created no edge. Fix: extract type annotation from `variable_declarator` nodes in the body walker, attributed to the enclosing symbol (no new nodes). Gated on `nodeType === 'variable_declarator' && TYPE_ANNOTATION_LANGUAGES.has(language)` → effectively **TS/TSX only** (Rust/Go/Java/C# use different AST shapes, e.g. Rust `let_declaration`/`type` field — verified Rust returns 0).
- **Imports/re-exports weren't dependencies.** A symbol imported and only re-exported / put in a registry array / passed as an arg / used in JSX created NO edge (only *called*/*instantiated*/*signature-typed* symbols linked). Fix in `src/extraction/tree-sitter.ts`: `emitImportBindingRefs` (per named/default/aliased binding, ~line 1750) called from `extractImport` (~line 1626, TS/JS gate); `emitReExportRefs` (per `export {X} from './y'`) called from a new `export_statement`-with-`source` dispatch branch (~line 381). Both push `imports` refs attributed to the **file node**; the resolver maps them to the definition via `resolveViaImport`. **This is the 62.5%→95.8% win.**
- **`TYPE_ANNOTATION_LANGUAGES`** (`src/extraction/tree-sitter.ts` ~line 2564): typescript, tsx, dart, kotlin, swift, rust, go, java, csharp.
- Full findings saved to memory: `~/.claude/projects/-Users-colby-Development-CodeGraph-codegraph/memory/impact-coverage-findings.md`.

## Gotchas
- **STALE INDEX is a trap.** The repo's checked-in `.codegraph/codegraph.db` was from Jun 2 and under-reported massively (types.ts showed 13 dependents; a fresh reindex gave 35, then 47 after fixes). **Always measure on a FRESH reindex into a temp dir**, never the repo's live `.codegraph`. (Also load-bearing for docker-app: it MUST reindex before computing impact.)
- **Node-count must stay stable** (no graph explosion) — the fixes add only edges, never nodes. Verify before/after. Resolution creates no nodes (confirmed); a deterministic ±1 from extraction-worker timing is noise.
- **Don't conflate same-basename files** when auditing (`src/types.ts` vs `src/resolution/types.ts` vs `src/ui/types.ts`). Match on full path.
- **Import-linking design differs by language:** named-symbol imports (Python `from x import Y`, Java `import com.x.Y`, Rust `use ...::Y`) port from the TS approach; **package/namespace imports (Go `import "pkg"`→`pkg.Func`, C# `using NS`) need a different design** (no per-binding names).
- **Known unfixed gap:** default-import-with-rename (`import Button from './x'` where the default export isn't named `Button`) doesn't link — `resolveViaImport` matches local name, no default-export tracking. Affects calls too (pre-existing), rare here. Out of scope unless prevalent in the next language.
- Work is on `main` and uncommitted — **branch before committing** (CLAUDE.md rule). Maintainer handles version bumps/releases.

## How to test & validate
- `npm test` → full suite, expect **1134 passed | 2 skipped** (was 1131 before the import-linking tests).
- `npx vitest run __tests__/extraction.test.ts -t "dependency linking"` → the 3 new import/re-export tests.
- `npx vitest run __tests__/graph.test.ts` → strengthened "File dependency analysis" (real cross-file assertions, not just `Array.isArray`).
- **Coverage probe recipe** (reuse for the next language; swap the glob ext):
  ```bash
  npm run build
  NEW=$(mktemp -d); cp -R <repo>/src "$NEW/src"   # or clone a real repo
  node -e "const {pathToFileURL}=require('node:url');(async()=>{const idx=await import(pathToFileURL(require('path').resolve('dist/index.js')).href);const CG=idx.default?.default??idx.default??idx.CodeGraph;const cg=CG.initSync('$NEW',{config:{include:['**/*.py'],exclude:[]}});await cg.indexAll();cg.resolveReferences();cg.destroy();})();" 2>&1 | grep -vE "Experimental|trace"
  # coverage:
  sqlite3 "$NEW/.codegraph/codegraph.db" "WITH src AS (SELECT DISTINCT file_path fp FROM nodes WHERE kind='file'), deps AS (SELECT tgt.file_path fp, COUNT(DISTINCT s.file_path) n FROM edges e JOIN nodes tgt ON tgt.id=e.target JOIN nodes s ON s.id=e.source WHERE e.kind!='contains' AND s.file_path!=tgt.file_path GROUP BY tgt.file_path) SELECT (SELECT COUNT(*) FROM src) files, (SELECT COUNT(*) FROM src WHERE COALESCE((SELECT n FROM deps WHERE deps.fp=src.fp),0)>0) with_deps;"
  # audit 0-dependent files: list them, classify imported-but-unlinked (real miss) vs not-imported (correct).
  ```
- **Per-language method:** small + medium + large real repos; for each, run the coverage probe, then a controlled mini-probe (write 2 files isolating each suspected gap — in-body type, value import, re-export) to see exactly which edges resolve. Fix extraction/resolution. Re-validate: coverage up, node count stable, full suite green, ≥1 dedicated test.

## Repo state
- branch `main`, last commit `629d847 fix(extraction): index Vue <template> component usages (#629 follow-up) (#659)`
- uncommitted (6 files): `CHANGELOG.md`, `__tests__/extraction.test.ts`, `__tests__/graph.test.ts`, `src/db/queries.ts`, `src/extraction/tree-sitter.ts`, `src/graph/queries.ts`
- CHANGELOG `[Unreleased] → Fixes` has 2 new bullets (the `affected` fix + the "recognize far more dependencies" completeness bullet).

## Open threads / TODO
- [ ] **Pick + do the next language** (Python / Java / Go / C# — see gotchas for import-model differences). Python = broadest reach, named imports port cleanly, but less typed. Java = most complete win (always-typed + named imports, enterprise/Spring). Go/C# need different import-linking design.
- [ ] Per-language in-body type annotations still open for Rust/Go/Java/C#/Kotlin/Swift/Dart (different AST shapes; TS only so far).
- [ ] Consider default-import-with-rename resolution (needs default-export tracking) if the next language uses it heavily.
- [ ] Port the TS fixes to `codegraph-pro` — user said they'll do it via upstream merge (don't touch pro).
- [ ] Decide whether/when to commit (on `main`, uncommitted; branch first).

## Recent transcript (oldest → newest)
### Turn 1 — "affected CLI under-reports; copy the fix from codegraph-pro; impact must work on every language"
- **User:** getFileDependents only follows imports edges, returns 0; said a fix exists in `codegraph-pro` to copy over.
- **Claude:** Diffed impact files between repos → **byte-identical**; pro's working diff was the already-merged #629 Svelte/Vue work, NOT an affected fix. Confirmed bug via live index (`imports` edges all same-file; 72/72 files under-reported to 0).
- **Outcome:** Implemented the `affected` fix fresh (`getDependentFilePaths`/`getDependencyFilePaths`), tests, CHANGELOG. Asked how to scope the broader cross-language effort + pro port.
### Turn 2 — user chose "Fix TS coverage first" + "I'll merge pro upstream"
- **Claude:** Audited → biggest factor was a STALE index (47.5%→62.5% on reindex). Found in-body type annotations dropped by `visitFunctionBody`; fixed + tested. Coverage 62.5%, node count stable.
- **Outcome:** TS in-body type fix landed; reported the stale-index trap + per-language nature.
### Turn 3 — "We should have 100% coverage not 62.5%"
- **Claude:** Explained 100% file-coverage is wrong (workers/barrels). Audited 0-dependent files → 42/45 were imported-but-unlinked (real misses). Root cause via clean probe: value imports / re-exports create no edge. Implemented `emitImportBindingRefs` + `emitReExportRefs`. **Coverage 62.5%→95.8%**; residual 5 all proven correct-zero. Full suite green (1134), tests added.
- **Outcome:** Import/re-export linking landed + validated. Updated memory.
### Turn 4 — "ok let's do the next language"
- **Claude:** Asked (AskUserQuestion) which language: Python/Java/Go/C#, with import-model rationale.
- **Outcome:** User **rejected** the question and ran `/handoff save`. → Next session: pick a language and start measuring (don't re-ask via tool).
