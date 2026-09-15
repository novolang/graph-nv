# graph-nv

A graph is a set of nodes and a set of edges joining them. It is the
data structure behind a dependency list, a road network, a state
machine and a social network. This package brings directed and
undirected graphs to novo-lang, with the traversals, the shortest paths,
three layout algorithms and a DOT export. Its reference for the
interface is the Rust crate [petgraph](https://docs.rs/petgraph), and
its layout vocabulary is
[NetworkX](https://networkx.org/documentation/stable/reference/drawing.html)'s.
The points a layout answers are
[geometry-nv](https://novo-lang.org/packages/geometry-nv)'s.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What the pieces are

A **node** is a point in the graph and an **edge** joins two of them. In
a **directed** graph an edge has a source and a target and may be
followed only one way. In an **undirected** graph it may be followed
either way.

A **weight** is whatever the caller attaches to a node or to an edge.
`GrGraph<N, E>` carries a node weight type and an edge weight type, and
nothing in this package interprets either. A weight can be a name, a
struct, or nothing at all.

An **index** is a node's or an edge's position in the graph, counted
from zero. Every function here takes and answers indices, never
references. A traversal answers a list of node indices.

**Adjacency** is, for each node, the list of edges that leave it and the
list that enter it. **CSR** is the compact way of writing that down:
three integer lists, holding where each node's neighbours start, how
many it has, and the neighbours themselves, one after another.

A **cycle** is a path that returns to where it started. A graph with no
cycle is **acyclic**, and only an acyclic directed graph has a
**topological order**: an order in which every node comes after all the
nodes with edges into it.

A **connected component** is a set of nodes that can all reach each
other ignoring edge direction. A **strongly connected component** is a
set in which every node can reach every other following the directions.
The **condensation** collapses each strongly connected component to one
node, and is always acyclic.

A **shortest path** is the path of least total cost between two nodes.
**Dijkstra's algorithm** finds it when no edge costs less than nothing.
**A\*** finds it faster when the caller can supply an estimate of the
cost remaining. A **minimum spanning forest** is the cheapest set of
edges that keeps every node connected to the ones it could reach.

A **layout** assigns a position to each node. Nothing here draws.

## Install

```
novo pkg add graph-nv
```

## Example

```novo
use std.list
use grbuild
use grwalk

// The node filter every traversal takes. This one keeps every node.
fn anywhere(node: Int) -> Bool
    true

// Four tasks and the order they must run in, as a directed graph. A
// node's weight is its name and an edge's weight is its cost.
fn pipeline() -> Result<GrGraph<Str, Float>, GrFault>
    let names = ["fetch", "build", "test", "ship"]
    let links = [GrEdge { source: 0, target: 1, weight: 1.0 },
                 GrEdge { source: 1, target: 2, weight: 1.0 },
                 GrEdge { source: 2, target: 3, weight: 1.0 }]
    grbuild.of_edges(GrDirected, names, links)

fn main() [io]
    match pipeline()
        Err(e) => println(e.message())
        Ok(g) =>
            println("${grbuild.node_count(g)} nodes, ${grbuild.edge_count(g)} edges")

            // An order in which every task follows the ones it depends on.
            match grwalk.topological_order(g)
                Err(e)    => println(e.message())
                Ok(order) => println("${list.len(order)} tasks in order")

            // Everything reachable from the first task, through the nodes
            // the filter keeps.
            match grwalk.bfs_order(g, 0, GrOutgoing, anywhere)
                Err(e)      => println(e.message())
                Ok(reached) => println("${list.len(reached)} reachable")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: graph-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `grbuild` | The graph: the constructors, adding and removing nodes and edges, the counts and lookups, the neighbours, the reverse, a subgraph, and the conversions to and from CSR. |
| `grwalk` | Breadth-first and depth-first order, breadth-first distances, a frontier-at-a-time walk, reachability, topological order, cycle detection, layers, connected and strongly connected components, the condensation, and a two-colouring. |
| `grpath` | Dijkstra from one start and from several, a point-to-point shortest path, A\*, the path and cost read out of a finished search, all-pairs distances, a minimum spanning forest, and the two helper projections. |
| `grlay` | Circular, layered and force-directed layout, the force parameters, a repeatable seed, crossing counts, the bounding box, scaling to fit, edge endpoints and an interpolation between two layouts. |
| `grdot` | Writing a graph in the DOT language into a cursor the caller sized, with or without positions, with or without clusters, and the identifier and label checks. |
| `grfault` | Every reason an operation refuses, as one enum with eleven variants, and the cycle rendered with the caller's own names. |

## How to choose an entry point

**Build with `grbuild.of_edges` when you already hold both lists**, with
`grbuild.of_csr` when you already hold an adjacency triple, and with
`directed` or `undirected` followed by `add_node` and `add_edge` when
you are building one piece at a time.

**There are two ways to remove a node, and the choice is real.**

| | `remove_node` | `remove_node_stable` |
| --- | --- | --- |
| What happens to other indices | the last node takes the removed one's index | nothing; the index becomes a hole |
| Cost | constant, plus the edges that touched it | proportional to the graph |
| What the caller must do | remap the two indices the answer names | iterate with `is_live` or `live_nodes` |

Take the stable one when something outside the graph is holding indices
across the change, such as a renderer between two frames. Take the cheap
one when the next step rebuilds its indices anyway.

**A traversal answers a whole list; `grwalk.bfs_step` answers one
frontier at a time.** Take the second when the list would not fit, or
when the walk must stop early.

**Depth-first order comes in two forms and both are published.**
Pre-order is what most callers mean. Post-order is what a topological
sort and the strongly-connected-component algorithm are built from.

**`grpath.shortest_path` stops as soon as the goal is settled**, where
`grpath.dijkstra` settles the whole graph and answers the distance to
every node. Take the second when you will ask about more than one goal.

## The rules a user needs

1. **An index is a position, not an identity.** Adding a node or an edge
   appends and leaves every existing index alone. Removing one does not.
   See the table above.
2. **`remove_node` names both indices that moved.** `GrRemoval` carries
   the index that was vacated and the index whose occupant now sits
   there, so a caller remaps exactly those two.
3. **Nothing here does arithmetic on a weight.** The one place a number
   is needed is the cost of an edge in a shortest path, and it arrives
   as a function `fn(E) -> Float` the caller supplies. A road network
   whose edges carry both a distance and a speed limit is one graph, and
   the two meanings of "shortest" are two calls over it.
   `grpath.unit_cost` is the projection for edges that carry no number,
   and `grpath.zero_heuristic` turns A\* into Dijkstra.
4. **Every traversal takes a node filter.** A search across the nodes
   one owner holds, or the modules that compile, or the roads that are
   open, is a filter on the frontier rather than a copy of the graph.
   `grwalk.all_nodes` is the honest "no filter" and `grwalk.no_nodes` is
   its opposite.
5. **The start is filtered too.** A search from a node the filter
   rejects answers an empty list, not that node on its own.
6. **A traversal answers a list, not a callback.** A callback would
   carry the caller's effects into these signatures, and this package
   declares none.
7. **A path that does not exist is an empty path, not a failure.** So is
   a walk from a node with no neighbours, and the components of an empty
   graph. A disconnected graph is an ordinary graph.
8. **A negative edge cost is refused.** Dijkstra's correctness rests on
   a settled node never getting cheaper, and a negative edge breaks that
   without saying so. `GrNegativeWeight` names the edge. Bellman-Ford is
   not here.
9. **A cycle is reported with its nodes.** `GrCyclic` carries the cycle,
   closing back on its first node, and `grfault.cycle_text` renders it
   with names the caller supplies. `grwalk.find_cycle` answers the same
   thing without provoking a failure.
10. **`grwalk.topological_order` breaks ties by ascending index**, so a
    build system's output does not change when a file is touched. It
    uses Kahn's algorithm.
11. **`grpath.minimum_spanning_forest` breaks ties by cost and then by
    ascending index**, so two runs give one answer. It uses Kruskal's
    algorithm, which answers a forest on a disconnected graph rather
    than refusing.
12. **`grwalk.strongly_connected` answers the components in reverse
    topological order of the condensation**, which is the order a caller
    resolving mutually recursive definitions wants. It uses Tarjan's
    algorithm.
13. **`grlay.force_directed` runs a fixed number of steps, not until it
    converges.** A layout is then a pure function of the graph, the seed
    and the step count, and two runs give one picture. Watch
    `grlay.total_displacement` fall and run more steps if you want more
    convergence.
14. **`grlay.seed_from_hash` hashes the node index.** No function in
    this package draws entropy, so a layout that seeded itself would be
    either irreproducible or secretly seeded from a constant.
15. **`grlay.layered` refuses a cyclic graph.** A layered drawing of a
    cycle has edges pointing backwards, which is a picture that
    disagrees with the data. Lay out `grwalk.condensation` instead,
    which is always acyclic.
16. **A\*'s heuristic must never overestimate the remaining cost.** An
    overestimate makes A\* answer a path that is not shortest, with
    nothing to say so. Whether a function overestimates is a property of
    the whole graph, so it is checked during the search:
    `GrHeuristicInadmissible` names the node, the estimate and the true
    cost. `grpath.heuristic_is_consistent` checks it up front.
17. **`grdot` writes into a cursor the caller sized.**
    `grdot.dot_bound` answers how many bytes to size it for, and
    `GrBufferTooSmall` carries both numbers when it is not enough.

## What is not included

- **Bellman-Ford and the Floyd-Warshall algorithm over negative
  weights.** See rule 8. `grpath.all_pairs` runs Dijkstra from each
  node, so it inherits the same restriction.
- **Maximum flow, matching and graph isomorphism.**
- **A DOT reader.** `grdot` writes only.
- **Drawing.** A layout answers positions. Take them to
  [svg-nv](https://novo-lang.org/packages/svg-nv) for a document, or
  round them to cells for a terminal. Drawing performs output, and
  nothing in this package does.
- **A microcontroller build.** Every structure here is a growable list
  and a traversal allocates a frontier. A device walking a fixed graph
  of twelve nodes writes the twelve lines. This package makes no device
  claim and ships no device probe.

## Related packages

- [geometry-nv](https://novo-lang.org/packages/geometry-nv) owns the
  point a layout answers and the vector arithmetic force-directed layout
  is made of. Declaring a second point type here would make every
  consumer convert between two identical values on the way to a canvas.
- [svg-nv](https://novo-lang.org/packages/svg-nv) turns positions and
  edge endpoints into a document.
- [plot-nv](https://novo-lang.org/packages/plot-nv) is the same division
  for charts: it computes the drawing operations and something else
  performs them.
- [ndarray-nv](https://novo-lang.org/packages/ndarray-nv) is where an
  adjacency matrix would live, for a caller who wants the linear-algebra
  view of a graph rather than the adjacency-list one.

## Tests

```bash
novo test tests/grbuild_tests.nv      # 9 tests: the graph and the two removals
novo test tests/grlay_tests.nv        # 6 tests: the three layouts and their determinism
```

petgraph is the reference for the index model, the two removal rules and
the direction parameter. The algorithms are the textbook ones, and the
suite asserts the choices that make an answer reproducible: Kahn's tie
rule, Kruskal's tie rule, and Tarjan's component order.

The suite asserts that appending leaves every index alone, that
`remove_node` names both indices it moved, that `remove_node_stable`
leaves a hole `is_live` reports, that a filtered search from a rejected
start answers nothing, that a missing path is an empty path rather than
a failure, that a negative cost is refused, that a cycle comes back with
its nodes, and that a layout run twice from one seed gives one answer.

The tests compile today and fail at run, each on the
`not implemented: graph-nv.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `grbuild.GrGraph`, `.GrEdge`, `.GrRemoval`, `.GrKind`, `.GrDirection` | declared |
| `grwalk.GrFrontier`, `grpath.GrPath`, `.GrDistances` | declared |
| `grlay.GrLayoutStep`, `.GrForceParams`, `grdot.GrDotOptions`, `grfault.GrFault` | declared |
| `grbuild.directed`, `.undirected`, `.with_capacity`, `.of_edges`, `.of_csr` | no |
| `grbuild.add_node`, `.add_edge`, `.remove_node`, `.remove_node_stable`, `.remove_edge` | no |
| `grbuild.set_node_weight`, `.set_edge_weight` | no |
| `grbuild.node_count`, `.edge_count`, `.index_bound`, `.is_live`, `.live_nodes`, `.is_directed` | no |
| `grbuild.node_weight`, `.edge_at`, `.edges_of`, `.neighbors`, `.undirected_neighbors` | no |
| `grbuild.has_edge`, `.edges_between`, `.degree`, `.reversed`, `.subgraph`, `.to_csr`, `.check_graph` | no |
| `grwalk.all_nodes`, `.no_nodes` | no |
| `grwalk.bfs_order`, `.bfs_order_from`, `.bfs_distances`, `.bfs_start`, `.bfs_step`, `.frontier_reached` | no |
| `grwalk.dfs_preorder`, `.dfs_postorder`, `.is_reachable` | no |
| `grwalk.topological_order`, `.has_cycle`, `.find_cycle`, `.layers` | no |
| `grwalk.components`, `.component_count`, `.component_nodes`, `.is_connected`, `.two_colouring` | no |
| `grwalk.strongly_connected`, `.scc_of_node`, `.condensation`, `.is_strongly_connected` | no |
| `grpath.zero_heuristic`, `.unit_cost`, `.straight_line`, `.heuristic_is_consistent` | no |
| `grpath.dijkstra`, `.dijkstra_from`, `.shortest_path`, `.a_star`, `.all_pairs` | no |
| `grpath.path_to`, `.cost_to`, `.was_reached` | no |
| `grpath.minimum_spanning_forest`, `.spanning_cost`, `.spanning_tree_from` | no |
| `grlay.default_force_params`, `.force_params_for`, `.seed_from_hash`, `.check_seed` | no |
| `grlay.circular`, `.circular_by_component`, `.layered`, `.reduce_crossings`, `.crossing_count` | no |
| `grlay.force_step`, `.force_directed`, `.total_displacement`, `.interpolate` | no |
| `grlay.layout_bounds`, `.fit_to`, `.edge_points` | no |
| `grdot.debug_options`, `.default_options`, `.dot_bound`, `.node_label` | no |
| `grdot.write_graph`, `.write_graph_with_layout`, `.write_clusters` | no |
| `grdot.is_bare_identifier`, `.escape_label`, `.bad_labels` | no |
| `grfault`'s eleven variants, `.no_such_node`, `.is_about_the_graph`, `.cycle_of`, `.cycle_text` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
