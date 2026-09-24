# src/xrAICore/Navigation — the graphs and the one search that walks them

> Two navigation graphs of very different shapes, the join between them, one search
> algorithm, and the pooled structures that let that search run inside a frame budget.

Part of [chapter 14](../README.md).

## What this directory is responsible for

Three separable things share this directory.

**The graphs.** A *level graph* is the fine navigation mesh of the loaded level: a sorted
grid of walkable cells, four-connected, each carrying its height, its floor plane and four
stored cover values. A *game graph* is the coarse cross-level graph: a few thousand vertices
with real world positions and real stored edge lengths, spanning the entire game, and it is
what the off-screen simulation moves entities along. A *cross table* joins them, naming for
every mesh vertex of the loaded level the game vertex that covers it. All three are shipped
prebuilt inside the level data and are read as memory images, so their byte layouts are
frozen. Beside them sits a fourth, unrelated graph type — a general-purpose mutable directed
graph, used when a graph is *built* at runtime rather than loaded.

**The search.** One uniform-cost loop, and one heuristic loop that is the same code with an
estimate added to each vertex's ordering key. Every search the AI ever runs is one of those
two, assembled with a different set of policies. The policies themselves are the sibling
[`PathManagers/`](PathManagers/README.md) directory; what lives here is the loop, the
storage it needs, and the object that owns the preallocated workspaces.

**The value type.** An entity's navigation location — which mesh vertex and which game
vertex it occupies — which is how nearly every AI position in the engine is stored, in
preference to a free coordinate.

## Where it sits

It rests on the core layer for the virtual filesystem and the chunked container format (the
graphs are files), on the math layer for positions and planes, and on the script engine for
the one graph type mods are allowed to see. It is the substrate under
[`PathManagers/`](PathManagers/README.md), under [`Components/`](../Components/README.md) —
the planner borrows this directory's search rather than writing its own — and under every
creature brain in chapter 24.

## The load-bearing ideas

**Two graphs, two coordinate systems, one join.** On the level mesh a position is a cell
index and the arithmetic is integer: all distances derive from one cell width, neighbours
are the four compass directions, and world coordinates are packed into the record. On the
game graph a position is a world point and the arithmetic is real: edges carry measured
lengths and the heuristic is straight-line distance. The cross table is the only thing that
can answer "which coarse vertex is this fine vertex in", and every promotion of an entity
from the coarse simulation into the loaded level goes through it.

**The three carry identity stamps and are checked against each other at load.** A level
mesh, a game graph and a cross table that do not agree are not a crash — they are creatures
walking through walls, silently, for the rest of the session. So the check is up front and
fatal. This is the single most important invariant in the chapter.

**The graphs are mapped, not parsed.** Both graphs and the cross table are read by pointing
at a file region; a mesh vertex is a packed record and the mesh has hundreds of thousands of
them. A rebuild may parse field by field instead, but the layout still has to be reproduced
exactly, because the data ships with the game. The level mesh additionally has more than one
shipped generation, and the load path converts older records forward into the current shape
so that everything above it sees one array of one record type.

**Nothing is allocated during a search.** The search's vertex records come from a
bump-allocated pool, its visited set is a table sized to the whole graph, and both are
allocated once when the engine starts. Starting a new search is a counter reset and a
generation increment — never a clear of hundreds of thousands of entries. That is the reason
these structures exist as separate files at all, and it is a frame-budget decision, not a
language one.

**A search ends and leaves nothing behind.** The three budgets a search carries — a cost
ceiling, an expansion ceiling, and a cap on how many vertices may be touched — are enforced
inside the loop, and exceeding any of them is reported as failure. There is no saved
frontier and no continuation: a caller who wants to spread work over frames re-issues the
whole search. The budgets are therefore sized to be affordable every frame, not to guarantee
an answer.

**The frontier has two shapes and the choice is the search's tie-breaking rule.** An exact
binary min-heap is used where cost is real-valued and precision matters. A bucketed ladder —
cost space sliced into a fixed number of buckets, each a sorted list threaded through the
open vertices themselves, with a cursor that only ever rises — is used on the level mesh,
where it makes insertion and extraction constant-time and makes vertices within a fraction
of a world unit of each other indistinguishable in priority. Everything past the ladder's
top falls into one unordered bucket.

**The visited set has two shapes too, for the same reason.** Where a vertex's identity is a
small dense integer (the mesh, the game graph) the visited set is a direct table with one
slot per vertex, stamped with the search's generation. Where it is not — the planner's world
states — it is a fixed hash table whose entries come from a ring, so that eviction is the
only cleanup that ever happens.

**The answer's shape is a policy, not a fixed choice.** A search can reconstruct its result
as a list of *vertices* (remember only each vertex's parent) or as a list of *edges*
(remember the edge taken as well). The planner needs the second, because a plan is a
sequence of actions and an action is an edge.

**Cover is stored, not computed.** Each mesh vertex carries four occlusion values, one per
compass direction, baked into the level. Cover in an arbitrary direction is an integration
of those four over a field of view. A rebuild that computes cover at runtime will not match
the shipped levels' behaviour, because the values were authored alongside the geometry.

## The twins

### The level mesh

| Twin | Role |
|---|---|
| [`level_graph.h`](level_graph.h.md) | The mesh's declaration: a sorted grid of walkable cells with four-way links, and the whole vocabulary of spatial questions the AI asks of it |
| [`level_graph.cpp`](level_graph.cpp.md) | Loading the mesh, and the question every AI query starts with — which cell is this world position standing on |
| [`level_graph_inline.h`](level_graph_inline.h.md) | The mesh's coordinate system: packing world positions into cell indices and back, cell containment, the accessibility fence, the straight-line walk |
| [`level_graph_space.h`](level_graph_space.h.md) | The frozen records: the mesh header, the packed vertex, and the two geometric shapes its queries are phrased in |
| [`level_graph_manager.h`](level_graph_manager.h.md) | Presents the vertex array as one record shape, converting older file generations forward on load |
| [`level_graph_vertex.cpp`](level_graph_vertex.cpp.md) | Walking the mesh along a straight line — can I get there, how far, which cells — and cover interpolated into an arbitrary direction |
| [`level_graph_vertex_inline.h`](level_graph_vertex_inline.h.md) | The mesh's geometry primitives: segment intersection, a cell's projected contour, nearest point on a contour, the cover integral |

### The cross-level graph and the join

| Twin | Role |
|---|---|
| [`game_graph.h`](game_graph.h.md) | The cross-level graph object: the loaded graph file plus the queries the alife simulation and the searches make of it |
| [`game_graph_inline.h`](game_graph_inline.h.md) | Reading the graph in place — edges through stored byte offsets — and binding the right cross table when a level loads |
| [`game_graph_space.h`](game_graph_space.h.md) | The frozen byte layout of the cross-level graph: header, vertex, edge, level record |
| [`game_graph_script.cpp`](game_graph_script.cpp.md) | The part of the navigation surface mods are allowed to see |
| [`game_level_cross_table.h`](game_level_cross_table.h.md) | The table joining every mesh vertex of a level to the coarse vertex covering it |
| [`game_level_cross_table_inline.h`](game_level_cross_table_inline.h.md) | Attaching the table to a file chunk or to a region inside the game graph, and answering by direct indexing |

### The runtime-built graph

| Twin | Role |
|---|---|
| [`graph_abstract.h`](graph_abstract.h.md) | The general-purpose mutable directed graph, for graphs built at runtime rather than loaded |
| [`graph_abstract_inline.h`](graph_abstract_inline.h.md) | Vertex and edge lifecycle with an incrementally kept edge count, and a chunked save/load |
| [`graph_vertex.h`](graph_vertex.h.md) | A vertex: a payload, an adjacency list, and back-references |
| [`graph_vertex_inline.h`](graph_vertex_inline.h.md) | Why the back-references exist — removal costs a vertex's degree, not the graph |
| [`graph_edge.h`](graph_edge.h.md) | An edge: a weight, a destination, optionally a payload |
| [`graph_edge_inline.h`](graph_edge_inline.h.md) | An edge's identity, for lookup, is its destination's |

### The search

| Twin | Role |
|---|---|
| [`dijkstra.h`](dijkstra.h.md) | The uniform-cost search and the four policies it is assembled from |
| [`dijkstra_inline.h`](dijkstra_inline.h.md) | The loop: expand the cheapest open vertex, relax its neighbours, stop on the goal, a budget, or an exhausted frontier |
| [`a_star.h`](a_star.h.md) | The heuristic search declared as the uniform-cost search plus a split of paid cost and estimated remainder |
| [`a_star_inline.h`](a_star_inline.h.md) | The estimate added to the ordering key, and the one branch deciding whether a settled vertex may be improved |
| [`graph_engine.h`](graph_engine.h.md) | The one object owning every search the AI runs, and the three assemblies it holds |
| [`graph_engine_inline.h`](graph_engine_inline.h.md) | Building the three workspaces, and the four steps every search entry point shares — bind a policy, run, time it, report |
| [`graph_engine_space.h`](graph_engine_space.h.md) | The scalar types every search is measured in, and the names of the planner's world-state vocabulary |
| [`data_storage_constructor.h`](data_storage_constructor.h.md) | The rule for stacking the four storage policies into one object and one flat per-vertex record |

### Frontier, visited set, pool, result

| Twin | Role |
|---|---|
| [`data_storage_binary_heap.h`](data_storage_binary_heap.h.md) | The exact frontier: a binary min-heap of open vertices keyed by cost |
| [`data_storage_binary_heap_inline.h`](data_storage_binary_heap_inline.h.md) | The heap, plus the guard that keeps a smallest-first insertion pattern from degenerating |
| [`data_storage_bucket_list.h`](data_storage_bucket_list.h.md) | The approximate frontier: cost space sliced into a fixed ladder of buckets |
| [`data_storage_bucket_list_inline.h`](data_storage_bucket_list_inline.h.md) | Sorted lists threaded through the vertices themselves, with a monotonically rising cursor |
| [`vertex_manager_fixed.h`](vertex_manager_fixed.h.md) | The direct visited set: one slot per graph vertex, valid only for the current generation |
| [`vertex_manager_fixed_inline.h`](vertex_manager_fixed_inline.h.md) | The generation counter that makes starting a search one increment instead of a full clear |
| [`vertex_manager_hash_fixed.h`](vertex_manager_hash_fixed.h.md) | The hashed visited set, for searches whose vertex identity is not a dense integer |
| [`vertex_manager_hash_fixed_inline.h`](vertex_manager_hash_fixed_inline.h.md) | A fixed table over a ring of index records, where eviction is the only cleanup |
| [`vertex_allocator_fixed.h`](vertex_allocator_fixed.h.md) | The search's vertex pool: a fixed array handed out by bumping a counter |
| [`vertex_allocator_fixed_inline.h`](vertex_allocator_fixed_inline.h.md) | Why a new search is a counter reset — no allocation during a search |
| [`vertex_path.h`](vertex_path.h.md) | The cheaper result record: parent links only, answer as a list of vertices |
| [`vertex_path_inline.h`](vertex_path_inline.h.md) | Reconstruction — sized in one pass, filled in reverse, delivered start-first |
| [`edge_path.h`](edge_path.h.md) | The richer result record: remember the edge taken as well, so the answer is a list of edges |
| [`edge_path_inline.h`](edge_path_inline.h.md) | Edge reconstruction, in either direction |

### The value type

| Twin | Role |
|---|---|
| [`ai_object_location.h`](ai_object_location.h.md) | An entity's position expressed as navigation rather than coordinates: which mesh vertex, which game vertex |
| [`ai_object_location_impl.h`](ai_object_location_impl.h.md) | The half that needs the graphs — validated writes, and turning an identity back into a graph record |
| [`ai_object_location_inline.h`](ai_object_location_inline.h.md) | The half that needs no graph definitions: initialization to invalid, and the two identity reads |

## Subdirectories

| Directory | Contents |
|---|---|
| [`PathManagers/`](PathManagers/README.md) | The search policies — what makes one search answer many different questions |
| [`PatrolPath/`](PatrolPath/README.md) | Authored waypoint graphs and the per-level registry that owns them |
