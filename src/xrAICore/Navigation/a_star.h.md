# src/xrAICore/Navigation/a_star.h

> Declares the heuristic search as the uniform-cost search plus a per-vertex split of paid cost and estimated remainder.

**Needs** — [`a_star_inline.h`](a_star_inline.h.md) · [`dijkstra.h`](dijkstra.h.md) · [`vertex_path.h`](vertex_path.h.md) · [`data_storage_constructor.h`](data_storage_constructor.h.md)
**Used by** — [`a_star_inline.h`](a_star_inline.h.md) · [`graph_engine.h`](graph_engine.h.md)
**Tier floor** — T2: a declaration of a parameterised algorithm.

## Purpose

Declares the surface implemented in [`a_star_inline.h`](a_star_inline.h.md). The heuristic
search takes exactly the same assembly parameters as the uniform-cost search it is built on —
cost type, frontier ordering, visited-set lookup, vertex pool, path record, iteration counter,
and the euclidian-metric promise — and adds two fields to each vertex's search record.

## State

Stateless as a file; the per-vertex record it contributes is listed under the exported units below.

## Exported units

- `AStarVertexData` — the mixin that gives each search vertex a paid cost `g` and an estimated
  remaining cost `h`, on top of the ordering key `f` and parent link the uniform-cost search
  already contributes. The invariant `f = g + h` is maintained by the search, not by the record.
- `Search(max_vertex_count)` — construct, sizing the vertex pool and visited-set table.
- `find(path_manager)` — run a search; see the inline twin.

**Notes** — the heuristic search *is* the uniform-cost search with one extra term, and the
engine expresses that by inheritance rather than by a second copy of the loop. A rebuild may
merge the two into one loop whose estimate function returns zero for the uniform-cost case; the
only thing that must survive is that the frontier orders by `g + h` while relaxation compares
on `g` alone. Comparing on the ordering key instead would make the search prefer paths that
merely *look* closer to the goal, which is a different and wrong algorithm.
