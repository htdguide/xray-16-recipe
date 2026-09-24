# src/xrAICore/Navigation/data_storage_binary_heap.h

> Declares the exact frontier ordering — a binary min-heap of open vertices keyed by cost.

**Needs** — [`data_storage_binary_heap_inline.h`](data_storage_binary_heap_inline.h.md) · [`xrCore/xr_types.h`](../../xrCore/xr_types.h.md)
**Used by** — [`data_storage_binary_heap_inline.h`](data_storage_binary_heap_inline.h.md) · [`graph_engine.h`](graph_engine.h.md)
**Tier floor** — T2: an ordering structure over preallocated slots.

## Purpose

Declares the surface implemented in
[`data_storage_binary_heap_inline.h`](data_storage_binary_heap_inline.h.md). This is one of the
two frontier orderings a search can be assembled with; it is the *exact* one — the vertex it
hands back is always the cheapest open vertex, with no approximation. The engine picks it for the
planner and for the string-keyed search, where the number of open vertices is small and the cost
values do not fall into a known range.

## State

Stateless. The frontier's state is in [`data_storage_binary_heap_inline.h`](data_storage_binary_heap_inline.h.md).

## Exported units

- `VertexData` — contributes no per-vertex fields. A heap keeps its order in its own array, so a
  vertex does not need to know where it sits. This is the heap's main difference from the bucket
  frontier, which threads links through the vertices themselves.
- `init` — begin a new search: the heap becomes empty.
- `is_opened_empty` — whether the frontier is exhausted.
- `add_opened(vertex)` — insert an open vertex.
- `decrease_opened(vertex, old_key)` — the vertex's cost has fallen; restore the ordering.
- `remove_best_opened` — drop the cheapest vertex from the frontier.
- `add_best_closed` — mark the cheapest vertex closed (it is still in the frontier at this point).
- `get_best` — the cheapest open vertex.

**Notes** — the heap array is sized once, at construction, to the maximum vertex count the search
was built for, and never grows. Overflowing it is a programming error, not a runtime condition:
the search's own visit budget is what keeps the frontier inside the array.
