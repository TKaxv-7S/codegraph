# Design + status: same-file value-reference edges

**Status:** SHIPPED (default-on for TS/JS/tsx + Go; `CODEGRAPH_VALUE_REFS=0` disables). The
emitter lives in `TreeSitterExtractor.flushValueRefs` (`src/extraction/tree-sitter.ts`).
**Motivation:** close the impact-analysis hole for *value consumers*. Static
extraction edges calls, imports, and inheritance, but never edges a constant to the
symbols that read it — so changing a config object / lookup table / shared constant
looked like "nothing depends on this." This is the "change this table, break its
readers" class of change (the ReScript-PR false positive that motivated the work).

---

## TL;DR for a new session

We emit a `references` edge (`metadata: { valueRef: true }`) from a reader symbol to
the **file/package-scope `const`/`var` it reads**, same-file only, for TS/JS/tsx + Go. Those edges
flow straight into `getImpactRadius` / `codegraph impact` and the impact trail in
`codegraph_explore` / `codegraph_node` — no agent-behaviour change required.

The win is **impact-radius correctness**, not agent read-reduction (see "Agent A/B").

## Edge semantics

- **Target:** a file-scope `const`/`var` whose name is "distinctive" (≥3 chars and
  contains an uppercase letter or `_`) — dodges the local-shadowing precision trap
  that single-letter / all-lowercase names invite.
- **Reader (source):** any `function` / `method` / `const` / `var` symbol whose body
  references the target name.
- **Same-file only** — resolution is unambiguous without import/scope analysis.
- **Deduped** per `(reader, target)`. **Additive** — adds edges, never nodes.

## Precision guards (in emission order)

1. **`isGeneratedFile(path)`** — skip suffix-recognised generated files (`.pb.ts`,
   `.min.js`, …). Path-only; it cannot catch content-minified bundles.
2. **Shadow prune (#895)** — drop any target whose name is bound by **more than one
   `variable_declarator`** in the file. A bundled/Emscripten `const Module` re-declared
   as an inner `var Module` / param resolves to the *inner* binding for nested readers,
   so a file-scope edge to it is a false positive. The inner re-declarations aren't
   extracted as graph nodes, so we count them at the syntax level. This is what catches
   the content-minified bundles guard #1 misses.
3. **Distinctive-name + same-file** as above.

## Validation matrix — TypeScript / JavaScript / Go

Method per repo: index the same tree twice (value-refs on vs `CODEGRAPH_VALUE_REFS=0`),
diff node/edge counts, spot-check precision, and measure `codegraph impact` on a few
file-scope consts. Node count must be **identical** on/off (edges-only feature).

**TypeScript**

| Repo | size | files | nodes (on=off) | +value-ref edges | precision | `impact` on→off example |
|---|---|---|---|---|---|---|
| sindresorhus/ky | small | 54 | 562 (stable) | +29 (0.8%) | all sampled TP | — |
| excalidraw/excalidraw | medium | 645 | 10,301 (stable) | +717 (1.6%) | TP after shadow prune (#895 removed 23 woff2-bundle FPs) | `tablerIconProps` 1→**170** |
| microsoft/vscode | large | 11,548 | 333,999 (stable) | +10,605 (0.69%) | all sampled TP; no param-shadow / bundle FPs in top 200 | `LayoutStateKeys` 1→**85**, `CORE_WEIGHT` 1→52 |

**JavaScript** (same extractor; CommonJS, `var`, IIFE/UMD)

| Repo | size | files | nodes (on=off) | +value-ref edges | precision | `impact` on→off example |
|---|---|---|---|---|---|---|
| expressjs/express | small | 147 | 1,082 (stable) | +27 (0.75%) | all sampled TP | — |
| eslint/eslint | medium | 1,420 | 7,167 (stable) | +1,192 (4.2%) | all sampled TP; guard holds; no minified-file FPs | `internalSlotsMap` 1→**32**, `INDEX_MAP` 1→27 |
| webpack/webpack | large | 9,371 | 28,922 (stable) | +3,521 (4.8%) | all sampled TP; guard holds; no minified-file FPs | `LogType` 1→**89**, `LOG_SYMBOL` 1→90, `UsageState` 2→52 |

**Go** (package-level `const`/`var`; required extending the shadow prune — see below)

| Repo | size | files | nodes (on=off) | +value-ref edges | precision | `impact` on→off example |
|---|---|---|---|---|---|---|
| gin-gonic/gin | small | 110 | 2,599 (stable) | +166 (1.9%) | all sampled TP; guard holds | `abortIndex` 1→**24**, `jsonContentType` 1→8 |
| gohugoio/hugo | medium | 952 | 19,160 (stable) | +1,616 (2.5%) | all sampled TP; guard holds | `filepathSeparator` 2→**26** |
| prometheus/prometheus | large | 1,329 | 23,322 (stable) | +3,466 (3.3%) | all sampled TP; guard holds | `rdsLabelInstance` 1→**82**, `ec2Label` 1→24 |
| kubernetes/kubernetes | very large | 19,160 | 251,086 (stable) | +20,574 (1.9%) | all sampled TP; guard holds on 250 targets | `KubeletSubsystem` 3→**138**, `LEVEL_0` 1→102 |

Across S/M/L in all three languages: node count never moved, the precision guards held, and
the `impact` OFF column is the bug — a const that 80–90 symbols read reports "1 affected"
without value-refs.

**Go required a code change** (unlike JS/tsx, which the existing guards covered unchanged).
Go puts its constants at package = file scope (good — the target gate fits), but its
declarators are `const_spec`/`var_spec`/`short_var_declaration`, not `variable_declarator`, so
the shadow prune was a no-op for Go and a package `const Timeout` shadowed by a local
`Timeout := …` produced a false positive. Extending the prune's declarator switch to Go's node
types fixed it (one synthetic repro, then clean across gin/hugo/prometheus). This is the
template for the next language: **the shadow prune is per-grammar and must be wired per
language** (see the playbook).

**`tsx` is covered by the TS rows** — excalidraw is a React/.tsx codebase, so the headline
`tablerIconProps` (1→170) and most of its targets live in `.tsx` files. The one
tsx-specific path — a const read *only* inside JSX (`<Foo x={CONST}/>`) — relies on the
reader-scan descending into the JSX subtree; it's locked by a unit test
(`value-reference-edges.test.ts`), so no separate tsx repo sweep is needed.

**JavaScript note — CommonJS `require` bindings are targets, and that's correct.** JS edge
growth (~4–5%) runs higher than TS (~0.7–1.6%) because `var x = require('…')` bindings and
module-level `var` state pass the distinctive-name gate and are read by same-file functions.
These are *not* noise: changing such a binding (swap the dependency, reassign the state)
genuinely affects its readers, so it's a legitimate impact target. Where it overlaps an
existing `calls` edge, `getImpactRadius` dedups by node — no double-counting. (TS `import`s
dodge this entirely: they're `import`-kind nodes, not `const`/`var`, so never targets.)

## Agent A/B — what it does and doesn't buy (excalidraw, sonnet/high, 12 runs)

- **Impact API (the win):** `impact` ON vs OFF — `tablerIconProps` 1→170,
  `COLOR_PALETTE` 15→26, `CaptureUpdateAction` 61→86. This is what `codegraph impact`
  and CodeGraph Pro's verdict engine consume via `getImpactRadius`.
- **Agent read-displacement: none — and that's expected.** On an indexed repo the agent
  answers impact questions in one codegraph call (0 Read / 0 Grep in *both* arms), and it
  reaches for `codegraph_search` / `callers`, **not** `impact`/`explore`, so it often
  doesn't query the value-ref edges at all. ON was never worse than OFF. **Do not claim
  value-refs reduces agent reads** — the win is blast-radius correctness, not fewer turns.
  (This is the "adapt the tool to the agent" wall: edges only help if the agent calls the
  edge-traversing tool.)

## Known limitations (intentional)

- **Parameter-only shadowing** is not guarded. The shadow prune counts
  `variable_declarator`s, so a file-scope const shadowed *only* by a function parameter of
  the same name would slip through. Not observed in S/M/L TS validation, and guarding it
  would over-prune legitimate consts whose name coincides with a parameter elsewhere in
  the file — so it's left unguarded until a real repo surfaces it.
- **Same-file only.** Cross-file value consumers (a const imported and read elsewhere) are
  not edged; that needs import/scope resolution and is out of scope.
- **Reactive/computed reads** (a value read only through a framework getter) have no static
  identifier to match and aren't covered.

## Extending to another language

The step-by-step runbook — wiring checklist, validation scripts, FP hunts, per-language
declarator types, and traps — is in
[`value-reference-edges-playbook.md`](./value-reference-edges-playbook.md). Point a fresh
session at it and say "Start on language X." In short: decide whether the language's
constants are file/module-scope (fits) or class-scope (bigger change); confirm the declarator
node type for the shadow prune; sweep small/medium/large public OSS repos; fix FP clusters;
add a matrix row here + a test.
