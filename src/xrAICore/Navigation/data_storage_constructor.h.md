# src/xrAICore/Navigation/data_storage_constructor.h

> The rule for stacking the four search policies into one storage object and one flat per-vertex record.

**Needs** — [`dijkstra.h`](dijkstra.h.md) · [`vertex_manager_fixed.h`](vertex_manager_fixed.h.md) · [`vertex_allocator_fixed.h`](vertex_allocator_fixed.h.md)
**Used by** — [`a_star.h`](a_star.h.md) · [`dijkstra.h`](dijkstra.h.md) · [`dijkstra_inline.h`](dijkstra_inline.h.md)
**Tier floor** — T2: a composition rule. It is pure structure; nothing here runs.

## Purpose

A search needs four independent services — a vertex pool, a visited-set lookup, a frontier
ordering, and a record of how the path was reached — and each of them needs fields on every
vertex. This file states how the four are combined: their per-vertex fields are merged into one
flat record, and their behaviours are stacked into one object in a fixed order.

A rebuild does not need this file as a file. It needs the two decisions in it.

## State

Stateless. It describes how other files' state is composed, and owns none itself.

## The two decisions

**One flat vertex record, not a chain of wrappers.** Each policy declares the fields it needs on
a vertex; the search vertex is the union of all of them, laid out contiguously. A search that
uses the bucket frontier carries bucket links and a generation stamp; one that uses the heap
carries neither. This matters because a level search touches tens of thousands of these records
inside one frame, and each is read and written several times — the record must be one cache line's
worth of adjacent fields, not a pointer chase across four objects.

**A fixed stacking order for the storage.** The composed storage is, innermost first:

```text
path record   (what to remember about how a vertex was reached)
  <- vertex pool      (where vertex records come from)
    <- visited-set lookup  (identity -> vertex record, per search generation)
      <- frontier ordering (which open vertex is cheapest)
```

Each layer calls down into the one below when its own operation needs it: adding a vertex to the
frontier also marks it open in the visited-set lookup, and resetting the frontier also resets the
pool and the lookup. The order is not arbitrary — the frontier must be outermost because it is
the layer the search talks to, and the pool must sit under the lookup because the lookup stores
references into the pool.

## `create_vertex`

**Contract** — take the next record from the pool and register it in the visited-set lookup under
the given vertex identity, returning it. This is the single seam where "allocate" and "record
that we have seen this identity" are fused; every other layer sees one operation. Does not place
the vertex in the frontier — the search does that separately, because a vertex is created before
its cost is known.

```text
FUNCTION create_vertex(vertex_id) -> Vertex
  record <- pool.create_vertex()               # next slot; no allocation
  RETURN lookup.create_vertex(record, vertex_id)
```

## `EmptyVertexData`

**Contract** — the neutral element: a policy that contributes no fields. Present so that a
search which wants no extra per-vertex data composes by the same rule as one that does.

**Notes** — `init` is forwarded down the whole stack on every new search, and that forwarding is
what makes a search cheap to start: nothing is allocated or cleared wholesale, each layer just
advances its own generation counter. The one exception is the generation counter wrapping, which
forces a full clear; see [`vertex_manager_fixed_inline.h`](vertex_manager_fixed_inline.h.md).
