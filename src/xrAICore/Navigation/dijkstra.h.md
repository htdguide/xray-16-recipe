# src/xrAICore/Navigation/dijkstra.h

> Declares the uniform-cost search and the four policies it is assembled from.

**Needs** — [`dijkstra_inline.h`](dijkstra_inline.h.md) · [`vertex_path.h`](vertex_path.h.md) · [`data_storage_constructor.h`](data_storage_constructor.h.md)
**Used by** — [`a_star.h`](a_star.h.md) · [`data_storage_constructor.h`](data_storage_constructor.h.md) · [`dijkstra_inline.h`](dijkstra_inline.h.md)
**Tier floor** — T2: a declaration of a parameterised algorithm; the tier is set by what it is assembled into, not by this file.

## Purpose

Declares the surface implemented in [`dijkstra_inline.h`](dijkstra_inline.h.md), and — the part
that matters to a rebuild — names the four independent choices a search is assembled from.
A search is not one class: it is a *cost type* plus a *frontier ordering* plus a *visited-set
lookup* plus a *vertex pool* plus a *what-to-remember-about-the-path* policy. The engine builds
three different searches from this same set (see [`graph_engine.h`](graph_engine.h.md)), and the
only reason they differ is which fillings they pick.

## State

Stateless as a file; the search's own state and the per-vertex record it contributes are in [`dijkstra_inline.h`](dijkstra_inline.h.md) and under the exported units below.

## The assembly parameters

- **cost type** — the scalar edge costs and accumulated distances are measured in. Real for
  navigation, a small integer for the planner.
- **frontier ordering** — how the open set is kept sorted: an approximate bucket list
  ([`data_storage_bucket_list.h`](data_storage_bucket_list.h.md)) or an exact binary heap
  ([`data_storage_binary_heap.h`](data_storage_binary_heap.h.md)).
- **visited-set lookup** — how a vertex identity maps to its search record: a direct table
  ([`vertex_manager_fixed.h`](vertex_manager_fixed.h.md)) or a hash chain
  ([`vertex_manager_hash_fixed.h`](vertex_manager_hash_fixed.h.md)).
- **vertex pool** — where search records come from
  ([`vertex_allocator_fixed.h`](vertex_allocator_fixed.h.md)).
- **path record** — whether reconstructing the answer needs only the vertices
  ([`vertex_path.h`](vertex_path.h.md)) or the edges taken
  ([`edge_path.h`](edge_path.h.md)).
- **iteration counter type** and whether the heuristic is **euclidian** — the latter is a promise
  about admissibility that [`a_star_inline.h`](a_star_inline.h.md) acts on.

## Exported units

- `Search` — the assembled search object, constructed with the maximum number of vertices it may
  ever visit. Owns its storage for its whole lifetime.
- `find(path_manager)` — run a search to completion; see the inline twin.
- `data_storage()` — hand out the storage, so a caller can tune the frontier's parameters before
  searching.

## The vertex record contributed here

```text
RECORD DijkstraVertexData
  f    : Distance          # cost of the best known path from the start to this vertex
  back : ref<Vertex>       # parent on that path; none for the start vertex
```

**Notes** — the search record for a vertex is assembled by mixing the fields each policy needs
into one flat record, so a vertex carries exactly the fields its particular search uses and
nothing more. In a rebuild this is a composition decision, not a language feature: what must
survive is that the per-vertex search state is *dense and preallocated*, because a path search
touches tens of thousands of vertices inside one frame.
