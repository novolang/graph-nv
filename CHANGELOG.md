# Changelog

All notable changes to graph-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.3 — 2026-09-28

The dependency ranges move to the dependencies' current releases.  A
pre-1.0 caret range admits only the release it names, so the old
ranges held this package on interface releases, and a program could
not take this package beside those packages' current releases.  No
signature in this package changed.

- geometry-nv: `^0.0.1` to `^0.1.0`.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `grbuild` — `GrGraph<N, E>` with node and edge weight parameters,
  adjacency the value owns, and two removals with two names because the
  index rule differs between them.
- `grwalk` — breadth first with a node filter, depth first in both
  orders, Kahn's topological sort, Tarjan's strongly connected
  components, connected components, layers and a two-colouring.
- `grpath` — Dijkstra, A\* with a caller-supplied heuristic, a minimum
  spanning forest, and the cost of an edge as a projection.
- `grlay` — circular, layered and force-directed layout, answering
  geometry-nv points and drawing nothing.
- `grdot` — DOT export into a cursor, with quoting refused rather than
  warned about.
- `grfault` — the few operations that can fail, with a cycle carried as
  its nodes.

### Known

- **`GrGraph<N, E>` is the load-bearing interface**, and an index is a
  position rather than an identity. `remove_node` swap-removes and
  names both indices that moved; `remove_node_stable` leaves a hole.
- **Every traversal takes a node filter**, and the start is filtered
  too.
- **An order is a list**, not a callback — which is what keeps this
  `core`.
- **Determinism is a promise**: Kahn, Kruskal and Tarjan all break ties
  by ascending index, the layout seed is a hash of the index, and
  force-directed layout runs a fixed step count.
- **A cycle is reported with its nodes.**
- **A negative edge weight is refused**, and Bellman-Ford is named as
  absent rather than approximated.
- **Layout answers points.** `std.canvas` is `[io]`, so drawing here
  would make the package `host`.
- The cost projection must be a **concrete** named function today; a
  generic one does not resolve when passed, which is filed against the
  toolchain.
- **No `@tier(embedded)` claim.** Every structure is a growable list.
- One dependency, geometry-nv, for the points.
