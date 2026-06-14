# Digital Abacus — Architecture

The Digital Abacus is an interactive environment for expressing and solving
algebraic relationships over the complex numbers. You build relationships as a
visual node graph and simultaneously watch them play out as points and linkages
on the complex plane. Drag any value and the rest of the graph rearranges itself
to keep every relationship true.

This document explains how the system is put together. For day-to-day
operational notes (commands, conventions, gotchas) see `CLAUDE.md`.

---

## 1. The big picture

```
                    ┌─────────────────────────────────────────────┐
                    │                  React UI                    │
                    │  digitalAbacus.tsx → Topbar / Sidebar /      │
                    │  CircuitBoard (reactflow) + Linkages (p5)    │
                    └───────────────┬──────────────┬──────────────┘
                                    │              │
                       reads nodes/wires      renders complex plane
                                    │              │
                    ┌───────────────▼──────────────▼──────────────┐
                    │              store.ts (valtio)               │
                    │  • holds the live graph (visibleGraph)       │
                    │  • runs the solver at 60fps                  │
                    │  • all mutations (add/select/clone/encap.)   │
                    │  • translates graph ⇄ reactflow shapes       │
                    └───────────────────┬──────────────────────────┘
                                        │
                    ┌───────────────────▼──────────────────────────┐
                    │          The model engine (src/model)         │
                    │                                               │
                    │  graph/   generic constraint hypergraph       │
                    │  coords/  complex-number specialization       │
                    │           (Coord, operations, edges)          │
                    └───────────────────────────────────────────────┘
```

The stack is Next.js 13 (pages router) + React 18 + TypeScript. The node graph
UI is `reactflow`; the complex-plane canvas is `p5` (via `react-p5`). Shared
state is a `valtio` proxy. Serialization and runtime validation use `zod`.

Almost all of the substance lives in `src/model/` (~11k of ~13k source lines).
The UI is comparatively thin: it renders the model and forwards user gestures
back into it.

---

## 2. The core abstraction: a bidirectional constraint hypergraph

Everything rests on a generic, value-agnostic layer in `src/model/graph/`.
It knows nothing about complex numbers — it could constrain any serializable
type `T`.

**`Vertex<T>`** (`graph/vertex.ts`) — holds a value of type `T`, an `id`
(`{ node, handle }`), and a list of dependencies `deps`. A vertex is *free*
(an input) when it has no dependencies and *bound* (an output) otherwise.

**`Edge<T>`** (`graph/edge.ts`) — groups several vertices and attaches a
`Constraint` relating them. This is a *hypergraph*: a single edge can touch
many vertices, not just two. The edge knows how to recompute its bound vertices
from its free ones (`update`) and how to flip an input/output relationship
(`invert`).

**`Constraint<T>`** (`graph/constraint.ts`) — the relation itself. Each
constraint declares, per position, whether that vertex is free or bound
(`getDependencies()` returns a boolean array), knows how to `update()` the bound
values from the free ones, and can `invert(take, give)` — swap which position is
the output. Key subclasses:

- **`EqualityConstraint`** — relates two vertices, copying a value from one to
  the other. Used for *wires*. Inverting simply flips the copy direction
  (`primaryLeft`).
- **`OperatorConstraint`** — an n-ary relation such as `a + b = c`, with one
  designated `bound` output position and an array of `ops` that can compute any
  single position from the others. Inverting changes which position is `bound`.

**`RelGraph<T>`** (`graph/relGraph.ts`) — the collection of vertices and edges,
plus the propagation logic:

- `update(iters)` runs every edge's constraint repeatedly. Because constraints
  are interdependent, multiple passes are needed for values to settle; passing a
  negative count runs to equilibrium (used carefully — it can diverge).
- `invert(take, give)` re-points dependencies, potentially across *several*
  edges, by walking the dependency tree (`_leafDeps` / `_intermedDeps`) to find
  a chain of single-step inversions. It includes loop detection and rolls back
  partial inversions on failure (`_invert`).

This is what makes the app feel like a solver. You never write `c = a + b`; you
assert the *relation* `a + b = c`, then drag any of the three points and the
other two move to keep it true. "Dragging the output to make it an input" is the
`invert` operation surfaced as a gesture.

---

## 3. Why complex numbers — and where the guarantees live

The complex-number choice is not cosmetic; it is what makes the inversion
machinery dependable. The generic graph is only as smooth as its constraints are
invertible, and over ℂ the elementary operations are *total and invertible* in
ways they are not over ℝ. Solutions are guaranteed to exist (the fundamental
theorem of algebra and friends), so "make any input the output" is always
meaningful.

**`Coord`** (`coords/coord/coord.ts`) is a complex number stored as cartesian
`(x, y)` but with polar accessors (`getR`, `getTh`). Multiplication, division,
exponentiation, and logarithm are implemented in polar form, so they stay clean
and are defined everywhere they can be.

The primitive operators (`coords/operations/ideal.ts`) each ship with *all*
their inversions, precisely because those inversions always exist over ℂ:

| Operator                  | Relation        | Inversions available                   |
|---------------------------|-----------------|----------------------------------------|
| `IdealComplexAdder`       | `a + b = c`     | subtract for `a` or `b`, add for `c`   |
| `IdealComplexMultiplier`  | `a · b = c`     | divide for `a` or `b`, multiply for `c`|
| `IdealComplexConjugator`  | `conj(a) = b`   | conjugate (self-inverse) either way    |
| `IdealComplexExponent`    | `e^a = b`       | `log` for `a`, `exp` for `b`           |

The only guarded case is division by zero (`IdealComplexMultiplier.accepts`
rejects a zero divisor). Multivalued operations are handled rather than avoided:
the exponential/log uses `_nearestN` to pick the branch of the (multivalued)
complex log nearest the current value, so dragging across a branch cut behaves
continuously.

`coords/coordVertex.ts` (`CoordVertex`) extends `Vertex` with display/interaction
state (dragging, hidden, selected, label) and the p5 drawing for a point.
`coords/coordGraph.ts` (`CoordGraph`) extends `RelGraph` with the complex-plane
specifics: building operator nodes, wires, reversal (inversion) gestures,
encapsulation state, and the per-frame `display()`.

---

## 4. Three capabilities layered on the core

### 4.1 Iterative vs. ideal solving

There are two solver modes (`settings.ts`; `UPDATE_MODE` defaults to
`UPDATE_ITERATIVE`).

- **Ideal** (`operations/ideal.ts`) — constraints snap directly to the exact
  answer each update.
- **Iterative** (`operations/iterative.ts`) — each output is nudged toward its
  target by at most `stepSize` per step, so points visibly glide into place and
  tangled, mutually-dependent constraints relax smoothly instead of jumping. The
  wire equality constraint has its own iterative variant
  (`IterativeComplexEqualityConstraint`) that "tracks" a moving target, with
  logic in `findApproachAngle` for chasing or fleeing a target that is itself in
  motion.

Mode selection happens in `getBaseConstraintByType` (`coords/edges/nodeEdge.ts`)
and `CoordGraph.getEqualityConstraintBuilder`.

### 4.2 Differentials (automatic differentiation)

`DifferentialCoord` (`coords/coord/differentialCoord.ts`) extends `Coord` with a
`delta` — a derivative carried alongside the value. Every operator implements
`updateDifferentials` with the real calculus (e.g. the multiplier applies the
quotient rule when solving for a divisor). When you grab a free point its delta
is set to `(1, 0)` and derivatives propagate through the whole graph, so the
system knows not just where each value is but how it is moving. `settings.showDifferentials`
renders them near the points.

Note: the standalone `operations/differential.ts` (a third "differential" mode)
is entirely commented out / dormant. The live differential behavior is the
`updateDifferentials` methods on the ideal/iterative operators.

### 4.3 Composites and encapsulation

A **`CompositeOperation`** (`operations/composites/compositeOperation.ts`) *is*
a `Constraint` whose behavior is an entire sub-`CoordGraph`. Its `update` pipes
incoming values into designated `interfaceVertexIds`, runs the inner graph, and
reads results back out. Because a composite is just another constraint, they
nest recursively, and inverting a composite delegates to inverting its inner
graph.

Built-in composites (`operations/composites/…`) — subtractor, divider,
reciprocal, exponent, nth-root, log, the trig/hyperbolic functions, and the
constants `i`, `e`, `π`, `φ`, plus a few applied examples (harmonic oscillator,
temperature, linear solver) — are pre-serialized sub-graphs. For example,
`subtractor.ts` is just an adder with a different vertex chosen as the output.
The `add*` functions are dispatched from `store.ts` `addNode`.

**Encapsulation** lets the user create their own composites (`store.ts`):
select some nodes, `startEncapsulation` works out the interface (which vertices
have wires crossing the selection boundary), and `commitEncapsulation` packages
the selected nodes + internal wires into a new `CompositeOperation`, replaces
them with a single node, and re-attaches the external wires. The
`ancestors` stack (`beginEditingComposite` / `endEditingComposite` /
`popToAncestor`) lets you descend into a composite to edit its internals and
climb back out.

---

## 5. Edge types

`coords/edges/` specializes the generic `Edge` for the complex plane:

- **`CircuitEdge`** — base class adding `selected` and complex-aware
  `updateDependencies` (with a special branch for `CompositeOperation` so a
  composite's internal dependency structure is reflected externally).
- **`NodeEdge`** — an operator node (primitive or composite). Holds `type`,
  `position`, `label`, `hidden`, builds the right constraint from serialized
  operator data, draws the operator's linkage on the plane (`displayLinkage`),
  and serializes/deserializes itself. This is the workhorse edge.
- **`WireEdge`** — an equality constraint connecting two vertices (a "wire" in
  the node UI). Tracks `source`/`target` and flips them on inversion.
- `compositeNodeEdge.ts` is fully commented out (dormant).

---

## 6. State and UI glue

**`store.ts`** (valtio) is the integration hub:

- Holds `visibleGraph` (the graph currently shown) and an `ancestors` stack for
  composite editing.
- Runs the solver on a `setInterval` at ~60fps, always updating the *outermost*
  graph so nested composites stay consistent.
- Exposes every mutation: `addNode`, `addWire`, `removeNode`, `removeWire`,
  `changeSelection`, `updateNodePosition`, `cloneSelected`, the encapsulation
  lifecycle, sticky-note editing, and labels.
- `useStore()` is the React hook that snapshots the graph and translates it into
  reactflow `nodes` and `wires` (and back via `edgeToNode` / `edgeToWire`),
  including the synthetic interface nodes shown while editing a composite.

**The two views:**

- `components/circuitBoard.tsx` — the reactflow node editor (nodes in
  `components/nodes/`, wires in `components/wires/`).
- `components/linkages.tsx` + `model/sketch.ts` + `model/graphics.ts` — the p5
  complex plane: grid and unit circle, draggable points, operator linkages, and
  pointer handling (`touchStarted/Moved/Ended`). `model/setup.ts` holds the
  shared `p` (p5 instance) singleton used throughout the model for math/drawing.

**Serialization & routing:** `model/deserializeGraph.ts` rebuilds a live
`CoordGraph` from serialized JSON — essential because composites are *stored* as
serialized sub-graphs and rebuilt on demand. The `pages/[serial]/index.tsx`
dynamic route can decode an entire serialized state from the URL (validated with
`serialStateSchema`), falling back to `DEFAULT_SERIAL_STATE`; `pages/index.tsx`
is the default empty board.

---

## 7. Directory map

```
src/
  model/
    graph/         generic constraint hypergraph (vertex, edge, constraint, relGraph)
    coords/
      coord/       Coord, DifferentialCoord (complex numbers)
      operations/
        ideal.ts        exact operators
        iterative.ts    stepwise operators + iterative wire equality
        differential.ts dormant (commented out)
        composites/     built-in composites + CompositeOperation
      edges/       CircuitEdge, NodeEdge, WireEdge
      coordGraph.ts, coordVertex.ts
    store.ts       valtio state + solver loop + all mutations
    settings.ts    solver mode, step size, scale, render toggles
    setup.ts       p5 instance singleton (p)
    sketch.ts      p5 pointer handlers + draw loop
    graphics.ts    grid / axes / readout drawing
    deserializeGraph.ts   JSON → live CoordGraph
    serialSchemas/ zod schemas for serialized graphs
  components/      React UI (digitalAbacus, circuitBoard, linkages, sidebar, topbar, nodes/, wires/)
  schema/          zod schemas for UI-level node/wire/handle/id
  lib/             canvas sizing, logger, initial data (mostly dormant)
  pages/           Next.js routes (index, [serial], _app, _document)
  styles/          global CSS
```

---

## 8. Mental model for adding features

- **A new primitive operation** → add an `OperatorConstraint` subclass in
  `ideal.ts` (and an iterative counterpart in `iterative.ts`), wire it into
  `OP_TYPE` and `getBaseConstraintByType` in `nodeEdge.ts`, add vertex defaults
  in `CoordGraph.addOperation`, and surface it in the sidebar UI.
- **A new built-in composite** → add an `add*` module under
  `operations/composites/` describing the sub-graph, register it in
  `BUILTIN_COMPOSITES` and in `store.ts` `addNode`, and add a sidebar entry.
- **New behavior on drag/solve** → look at the 60fps loop in `store.ts`, the
  per-frame `CoordGraph.display`/`update`, and the relevant constraint's
  `update` / `updateDifferentials`.
- **The subtle parts** to respect: the dependency bookkeeping
  (`Edge.updateDependencies`, `Vertex.deps`) and multi-step inversion
  (`RelGraph._invert`). These are the most likely to break when extending.

See `CLAUDE.md` for known rough edges and conventions.
