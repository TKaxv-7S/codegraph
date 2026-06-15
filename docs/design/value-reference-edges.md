# Design + status: same-file value-reference edges

**Status:** SHIPPED (default-on for TS/JS; `CODEGRAPH_VALUE_REFS=0` disables). The
emitter lives in `TreeSitterExtractor.flushValueRefs` (`src/extraction/tree-sitter.ts`).
**Motivation:** close the impact-analysis hole for *value consumers*. Static
extraction edges calls, imports, and inheritance, but never edges a constant to the
symbols that read it — so changing a config object / lookup table / shared constant
looked like "nothing depends on this." This is the "change this table, break its
readers" class of change (the ReScript-PR false positive that motivated the work).

---

## TL;DR for a new session

We emit a `references` edge (`metadata: { valueRef: true }`) from a reader symbol to
the **file-scope `const`/`var` it reads**, same-file only, for TS/JS/TSX. Those edges
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

## Validation matrix — TypeScript/JavaScript

Method per repo: index the same tree twice (value-refs on vs `CODEGRAPH_VALUE_REFS=0`),
diff node/edge counts, spot-check precision, and measure `codegraph impact` on a few
file-scope consts. Node count must be **identical** on/off (edges-only feature).

| Repo | size | files | nodes (on=off) | +value-ref edges | precision | `impact` on→off example |
|---|---|---|---|---|---|---|
| sindresorhus/ky | small | 54 | 562 (stable) | +29 (0.8%) | all sampled TP | — |
| excalidraw/excalidraw | medium | 645 | 10,301 (stable) | +717 (1.6%) | TP after shadow prune (#895 removed 23 woff2-bundle FPs) | `tablerIconProps` 1→**170** |
| microsoft/vscode | large | 11,548 | 333,999 (stable) | +10,605 (0.69%) | all sampled TP; no param-shadow / bundle FPs in top 200 | `LayoutStateKeys` 1→**85**, `CORE_WEIGHT` 1→52, `CONTEXT_FOLDING_ENABLED` 1→22 |

Across S/M/L: node count never moved, edge growth ≤1.6%, and the precision guards held
(the only false positives — excalidraw's 23 — were a single bundled file, fixed by the
shadow prune). The `impact` OFF column is the bug: a const that 85 symbols read reports
"1 affected" without value-refs.

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

1. Add the language to `VALUE_REF_LANGS` and confirm its declarator node type is
   `variable_declarator` (or adjust the shadow-prune scan for the grammar's equivalent).
2. Run the validation matrix above on small/medium/large real repos (public OSS only).
3. Hunt FPs: bundled/generated files, intra-file shadowing, param reuse. Fix clusters;
   record singletons. Add a row to the matrix.
