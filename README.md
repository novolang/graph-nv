# graph-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

petgraph's shape in novo-lang: directed and undirected graphs over node
and edge indices with node and edge **weight type parameters**,
adjacency as compact lists the value owns, the traversals and shortest
paths, and layout as pure arithmetic over geometry-nv points that a
canvas later draws.

- `grbuild` — the graph, and the index stability rules stated rather
  than implied;
- `grwalk` — breadth first, depth first, topological order, strongly
  connected components, connected components;
- `grpath` — Dijkstra, A\*, and a minimum spanning forest;
- `grlay` — circular, layered and force-directed layout, as points;
- `grdot` — DOT export into a cursor;
- `grfault` — the few things that can fail, and the cycle carried with
  its nodes.

```
novo pkg add graph-nv
novo pkg build
novo test
```

## The one example that will work

```novo ignore
use grwalk

// Which territories a faction still supplies: reachable from its
// headquarters, across territories it owns.
fn supplied(g: GrGraph<Str, Float>, hq: Int, mine: fn(Int) -> Bool)
    -> Result<[Int], GrFault>
    grwalk.bfs_order(g, hq, GrOutgoing, mine)
```

## The load-bearing interface: `GrGraph<N, E>`, and its indices

```novo ignore
pub struct GrGraph<N, E>
    kind: GrKind
    nodes: [N]
    edges: [GrEdge<E>]
    out_edges: [[Int]]
    in_edges: [[Int]]
    live: [Bool]
```

**An index is a position, not an identity.** That is what makes a graph
library usable from a language with no pointers: a traversal answers
`[Int]`, a shortest path answers `[Int]`, a layout answers a point per
index, and none of them hands back a reference into a structure the
caller also owns.

The cost is that indices move, and **every graph library that hides that
produces a bug nobody can see**. So the rule is in the surface:

| | |
| --- | --- |
| `add_node` / `add_edge` | append. Every existing index is unchanged. |
| `remove_node` | **swap-removes.** The last node takes the removed one's index. O(1) plus the touched edges, and the answer **names both indices that moved**. |
| `remove_node_stable` | leaves a **hole**. Every other index is unchanged; `index_bound` is then larger than `node_count`, and `is_live` / `live_nodes` are how a caller iterates. |

Two removals with two names, because the choice is real and a library
that picked one picks wrongly for half its callers: a renderer holding
indices across a frame needs the stable one, and a batch algorithm that
rebuilds its indices anyway should not pay for the holes. `remove_node`
is the cheap one and has the shorter name, which is the wrong way round
for safety — so `GrRemoval` answers *which* index moved rather than
leaving a caller to work it out.

## A weight type parameter, and it works

`GrGraph<N, E>` carries a node weight and an edge weight, and **nothing
here does arithmetic on either**. A weight is an opaque payload; the one
place a number is needed — the cost of an edge for a shortest path —
arrives as a named function `fn(E) -> Float` the caller supplies.

That is what keeps the type parameters free of a numeric bound. novo-lang
has no blanket impls, so an `E: Numeric` bound would mean one impl per
weight type in every consumer. It is also what a caller wants: a road
network whose edges carry a distance *and* a speed limit is one graph,
and `dijkstra(g, s, by_distance)` and `dijkstra(g, s, by_time)` are two
calls over one structure rather than two graphs.

The one thing to know: the projection must be a **concrete** named
function today. A generic one — `grpath.unit_cost<E>` is published for
that position — does not resolve when passed, and the tests here use a
locally declared `fn cheap(w: Float) -> Float` instead. In practice a
projection is domain-specific anyway, so that is the spelling a real
caller writes.

## Every traversal takes a node filter, and that is not a convenience

```novo ignore
pub fn bfs_order<N, E>(g: GrGraph<N, E>, start: Int, d: GrDirection,
                       keep: fn(Int) -> Bool) -> Result<[Int], GrFault>
```

The first real consumer of this package walks a graph of territories and
asks which are reachable from a headquarters **across territories one
faction owns** — a search whose frontier is filtered by a predicate the
graph knows nothing about. Every graph walk in a real program is that
walk: reachable through nodes that are still installed, through modules
that compile, through roads that are open.

Without the filter a caller builds a subgraph first, which allocates a
copy of a structure it is about to walk once. `all_nodes` is the honest
"no filter" and is published so nobody writes it; `no_nodes` is its
opposite, for a test and for a caller disabling a walk without a branch
at the call site.

**The start is filtered too.** A search from a node the filter rejects
answers an empty list, not that node alone — which is the answer a
caller filtering by ownership wants when the headquarters has fallen.

## An order is a list, and that is what keeps this `core`

A traversal answers `[Int]`. No callbacks, no visitor, no iterator
protocol — because a callback would carry the caller's effects into
these signatures, and a `core` package would then be `[io]` because
somebody printed while walking. A caller that cannot afford the list has
`bfs_step`, which is the same walk one frontier at a time.

Depth-first order is **two** answers and both are published: pre-order
is what a caller means by "depth first", and post-order is what a
topological sort and Tarjan's algorithm are built from. A library that
answered only the first makes its caller reimplement the second.

## Determinism is a promise, everywhere it can be

| | |
| --- | --- |
| `topological_order` | Kahn, ties broken by **ascending index** — so a build system's output does not change when a file is touched |
| `minimum_spanning_forest` | Kruskal, ties by cost then ascending index — two runs, one answer |
| `strongly_connected` | Tarjan, and the components come out in **reverse topological order of the condensation**, which is what a caller resolving mutually recursive definitions wants |
| `seed_from_hash` | a hash of the node **index**, not a random draw — a `core` package has no generator, and a layout that seeded itself would be either irreproducible or secretly seeded from a constant |
| `force_directed` | a **fixed step count**, not a convergence tolerance |

That last one is the layout argument. The usual formulation runs until
it converges, so the answer depends on a tolerance, the node count and
floating-point rounding — two runs over one graph give two pictures and
a caller diffing them sees noise. Here every step is the same
arithmetic and the layout is a pure function of `(graph, seed, steps)`.
A caller that wants convergence runs more steps and watches
`displacement` fall.

## A cycle is reported with its nodes

`GrCyclic` carries the cycle, closing back on its first node, and
`cycle_text` renders it with the caller's own names. *"This graph has a
cycle"* sends a caller looking for it; `a -> b -> c -> a` does not, and
finding it again afterwards is a second traversal nobody should have to
write. `find_cycle` answers the same thing without provoking a failure,
for a lint or a diagnostic.

Most operations **cannot** fail, and the surface says which can. `bfs`
from a node with no neighbours is that node; `components` of an empty
graph is an empty list; a shortest path that does not exist is an
**empty path**, not a fault — a disconnected graph is an ordinary
graph. Modelling those as failures would make the ordinary case travel
the failure path.

A negative edge weight **is** refused: Dijkstra's correctness rests on a
settled node never getting cheaper, and a negative edge breaks that
silently — the answer comes back, it is wrong, and nothing says so.
`GrNegativeWeight` names the edge. Bellman-Ford is not here, and it is
named as absent rather than approximated by running Dijkstra anyway.

## Layout is numbers, and drawing them is somebody else's `[io]`

The plan's note for this row is *"traversal and layout with no canvas
attached"*, and that is a **layer** decision as much as a scoping one:
`std.canvas` is `[io]`, so a `layout` that drew would make this package
`host` and take it off every consumer that only wanted a topological
sort. plot-nv made the same decision for the same reason.

It is also what makes a layout testable and replayable. A test asserts
positions; a renderer replays them; a caller that wants an SVG hands
them to svg-nv and one that wants a terminal rounds them to cells.

Three layouts, one per shape of graph: `circular` for a graph with no
structure worth showing (and the seed the others start from), `layered`
for a DAG, `force_directed` for everything else. `layered` **refuses** a
cyclic graph rather than laying it out anyway, because a layered
drawing of a cycle has edges pointing backwards — a picture that lies
about the data. A caller that has one lays out `grwalk.condensation`,
which is always acyclic.

## What sector would take, and what it keeps

`orbit/sector` is the dogfood package this row is split out of, and what
it actually contains is smaller than the row implies. `sim.nv` holds a
CSR adjacency triple — `sec_adj_off`, `sec_adj_len`, `sec_adj` — built
once by a shared-vertex join, and `recompute_supply` is a **filtered
breadth-first search** over it from each faction's headquarters across
the sectors that faction owns. `influence.nv` folds one integer field
over the same adjacency.

**What it would take:**

- the CSR triple, which is why `of_csr` and `to_csr` are published on
  the boundary rather than hidden — sector already has the three lists
  and should not rebuild a graph node by node out of them;
- `bfs_order_from` with the ownership predicate as `keep`, which is
  `recompute_supply` minus its loop. That function **is** why every
  traversal here takes a filter.

**What it keeps, and why each one is sector's rather than a graph
library's:**

- the **adjacency rule**. Two sectors are neighbours when their rings
  share a vertex within half a metre. That is a fact about a hand-drawn
  map, not about graphs, and it stays in `with_sectors`.
- the **influence weights** — the district value, the contact bonus, the
  supply-cut penalty. A weighted fold over neighbours is three lines
  over `neighbors`; the weights are the game.
- the **GPU leg**. `influence.nv` runs the same field as a WGSL kernel
  and asserts bit-identity against the CPU fold. That pair is sector's
  whole reason for having the code, it needs `[io]` for the GPU, and it
  could not live in a `core` package at all.
- the **integer arithmetic**. Sector's field is `i32`-safe integers so
  that the two legs are exactly equal by construction. This package's
  costs are `Float`.

So the split is: sector keeps its map, its weights and its kernel, and
takes the walk. Two functions, and the reason it is worth it is that the
walk it has today is written inline inside a supply-recomputation and
cannot be tested on its own.

## The layer

`core` — no effects. A graph is a list of nodes, a list of edges and
arithmetic over them; a layout answers points; DOT is written into a
cursor the caller sized.

**No `@tier(embedded)` claim, and none is intended.** Every structure
here is a growable list and a traversal allocates a frontier. A device
that walks a fixed graph of twelve nodes writes twelve lines rather than
taking this. The audit's `core-embedded` row passes as *makes no device
claim*.

One dependency, **geometry-nv**, for the points layout answers in.
`GeomPointF` is a `@value` struct of two floats in a `core` package that
already owns the vector arithmetic force-directed layout is made of, and
declaring a second point type here would make every consumer convert
between two structurally identical values on the way to a canvas.

## Where the names come from

Every public type and every enum variant is prefixed `Gr`. `Graph`,
`Edge`, `Path`, `Direction` and `Fault` are all names another package
would want, and struct identity is keyed by name across a whole program.
`GrGraph` stutters slightly and is unambiguous, which is the trade the
naming rule asks for.

## The reference implementation

petgraph for the API shape — the index model, the two removal rules, the
direction parameter — and networkx for the layout vocabulary. The
algorithms are the textbook ones and are named where the choice between
two matters: Kahn rather than a DFS reversal for the topological sort
(determinism), Tarjan rather than Kosaraju for the strongly connected
components (one pass), Kruskal rather than Prim for the spanning forest
(it answers a forest on a disconnected graph rather than failing).

Left out and named above: Bellman-Ford, maximum flow, matching,
isomorphism, and a DOT **reader**.

## Status

Interface only. `novo pkg build` is clean, `novo doc` renders, the API
tests under `tests/` are red against `todo()` bodies, and the three
shard rows — `effect-budget`, `dep-layer`, `no-discharge-in-core` — are
green. The first implementation is the `0.1.0` published over this.
