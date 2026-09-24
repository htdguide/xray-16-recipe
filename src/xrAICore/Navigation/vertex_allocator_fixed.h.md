# src/xrAICore/Navigation/vertex_allocator_fixed.h

> Declares the search's vertex pool — a fixed array of records handed out by bumping a counter.

**Needs** — [`vertex_allocator_fixed_inline.h`](vertex_allocator_fixed_inline.h.md) · [`xrCommon/xr_vector.h`](../../xrCommon/xr_vector.h.md)
**Used by** — [`data_storage_constructor.h`](data_storage_constructor.h.md) · [`graph_engine.h`](graph_engine.h.md) · [`vertex_allocator_fixed_inline.h`](vertex_allocator_fixed_inline.h.md)
**Tier floor** — T1: its whole point is that no allocation happens during a search, which is a hard frame-budget requirement.

## Purpose

Declares the surface implemented in
[`vertex_allocator_fixed_inline.h`](vertex_allocator_fixed_inline.h.md). A search may visit tens
of thousands of vertices inside one frame, and each visit needs a record; allocating those one at
a time would put the frame budget at the mercy of the allocator. So the pool is sized once, at
search construction, and a search never allocates.

The pool size is a compile-time property of the assembled search, not a runtime argument: the
navigation search reserves 65536 records in the game and two million in the offline level
compiler, the planner 8192, the string search 1024.

## State

Stateless. The pool's state is in [`vertex_allocator_fixed_inline.h`](vertex_allocator_fixed_inline.h.md).

## Exported units

- `VertexData` — contributes no per-vertex fields; the pool needs nothing recorded on a vertex.
- `init` — begin a new search: the hand-out counter returns to zero and every record in the pool
  becomes available again. Nothing is cleared, because a reused record is fully overwritten
  before it is read.
- `create_vertex` — hand out the next record.
- `get_visited_node_count` — how many records have been handed out this search. This is one of
  the three ceilings a search's budget is checked against.

**Notes** — the reserve is a hard ceiling with no fallback: exceeding it is a programming error,
caught by an assertion, not a condition handled at runtime. What keeps a search inside it is the
search's own visit budget, which the caller sets below the reserve. A rebuild that grows the pool
instead must accept that growth can happen mid-frame.
