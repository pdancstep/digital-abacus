# CLAUDE.md

Operational guide for working in this repo. Read `ARCHITECTURE.md` for the full
design; this file is the short, practical reference.

## What this is

The Digital Abacus: a Next.js + React app for building and solving algebraic
relationships over the complex numbers. Relationships are a visual node graph
(reactflow) mirrored on a complex-plane canvas (p5). Dragging any value re-solves
the rest of the graph. The engine lives in `src/model/`.

## Commands

```bash
npm install        # first-time setup
npm run dev        # dev server (next dev, piped through pino-pretty)
npm run build      # production build — use this to check it compiles
npm run start      # serve the production build
npm run lint       # next lint
```

- There is **no test suite** (no jest/vitest/playwright). Verify changes with
  `npm run build` and by exercising the app in the browser. Consider adding tests
  for the model engine — it is pure and very testable.
- Node 18-era toolchain (`@types/node` 18). No `.nvmrc`; use Node 18 LTS.
- Path alias: `@/` → `src/` (see `tsconfig.json`).

## Where things live (start here)

- `src/model/graph/` — generic constraint hypergraph (`vertex`, `edge`,
  `constraint`, `relGraph`). Value-agnostic core.
- `src/model/coords/` — complex-number specialization: `coord/` (the numbers),
  `operations/` (operators + composites), `edges/` (`NodeEdge`, `WireEdge`).
- `src/model/store.ts` — valtio state, the 60fps solver loop, **all** mutations,
  and graph ⇄ reactflow translation. The integration hub.
- `src/model/settings.ts` — solver mode (`UPDATE_MODE`), `stepSize`, scale,
  render toggles.
- `src/components/` — the React UI.

## Conventions

- TypeScript throughout; serialization/validation via `zod` (`serialSchemas/`,
  `src/schema/`).
- State is a `valtio` proxy (`store.ts`). Mutate through the exported functions,
  not by reaching into the graph from components.
- The p5 instance is a module singleton `p` in `model/setup.ts`, imported
  wherever drawing or p5 math is needed. It is `null` until the canvas mounts —
  hence the `p!` non-null assertions everywhere.
- `Coord` holds cartesian `(x, y)`; use the polar accessors (`getR`, `getTh`)
  and the provided arithmetic methods (`translate`, `multiply`, `divide`, `exp`,
  `log`, …) rather than recomputing by hand. `mut_*` variants mutate in place.
- New operators come in pairs: an `ideal` version and an `iterative` version.

## Gotchas / known rough edges

- **`src/model` is excluded from typechecking.** `tsconfig.json` has
  `"exclude": ["node_modules", "src/model"]` (and oddly `include`s a stray
  `src/model/graph/edge.js`). This means the most important code is *not*
  type-checked by the build. Be extra careful with types in the model, and
  consider fixing this config early.
- **Lots of dormant code.** `operations/differential.ts` and
  `edges/compositeNodeEdge.ts` are fully commented out; URL save/load in
  `store.ts` (`useStore` effects) is disabled; `lib/data/initialData.ts` is
  mostly commented out. Don't assume commented blocks reflect current behavior.
- **Stray `console.log`s** sit in hot paths (the solver, dependency updates,
  selection). Clean up as you touch them.
- **The subtle machinery:** dependency bookkeeping (`Edge.updateDependencies`,
  `Vertex.deps`) and multi-step inversion (`RelGraph._invert`) are the trickiest
  parts and the most likely to break when extending. There is a self-described
  "big hack" in `updateDependencies` (vertices depend on themselves) — understand
  it before changing dependency logic.
- **Solver stability:** `RelGraph.update(-1)` runs to equilibrium and can diverge
  if the graph is unstable. The live loop uses a fixed `settings.updateCycles`
  for safety.
- **README task list** flags open work: "schema broadcasting to instances" and
  "lock people out of deleting during encapsulation."

## How to add things

- **New primitive operator:** subclass `OperatorConstraint` in `ideal.ts` +
  iterative version in `iterative.ts` → register in `OP_TYPE` and
  `getBaseConstraintByType` (`coords/edges/nodeEdge.ts`) → add vertex defaults in
  `CoordGraph.addOperation` → add a sidebar entry.
- **New built-in composite:** add an `add*` module under
  `operations/composites/` (a serialized sub-graph) → register in
  `BUILTIN_COMPOSITES` (`compositeOperation.ts`) and in `store.ts` `addNode` →
  add a sidebar entry.
- **UI change:** components are thin views over `useStore()`; push logic into
  `store.ts` rather than into components.

## Git

- `origin` → Paul's fork (`pdancstep/digital-abacus`).
- `upstream` → original (`dnsosebee/digital-abacus`); pull from here for upstream
  changes.

## Working agreements

- Match existing style; prefer small, reviewable changes.
- After non-trivial changes, run `npm run build` (the closest thing to a test
  gate) before considering the work done.
- Keep `ARCHITECTURE.md` and this file updated when the structure changes.
