# src/xrAICore/Navigation/vertex_allocator_fixed_inline.h

> Hand out search vertex records by bumping a counter into a preallocated array; a new search just resets the counter.

**Needs** — [`vertex_allocator_fixed.h`](vertex_allocator_fixed.h.md)
**Used by** — [`vertex_allocator_fixed.h`](vertex_allocator_fixed.h.md)
**Tier floor** — T1: no allocation during a search is the requirement this file exists to meet.

## Purpose

Hands out the per-vertex search records. The whole file exists to make that free: the records are allocated once for the life of the search object, and starting a new search is one assignment.

## State

```text
RECORD VertexPool
  records : list<Vertex>   # fixed length = the reserve; allocated once, at construction
  handed  : int            # how many have been handed out this search
```

**Invariants** — `handed` is the count of live records this search, and also the index of the
next one to hand out. Records below `handed` belong to the current search; records above are
stale content from previous searches and are never read before being fully written. Between
searches `handed` is deliberately set to "impossible" rather than zero, so that handing out a
record before a search has been initialized trips an assertion instead of quietly returning
record zero.

## `create_vertex`

**Contract** — return the next record in the pool and advance the counter. Constant time, no
allocation, no clearing. Overflowing the reserve is a programming error: the search's visit
budget is what must keep this from happening.

```text
FUNCTION create_vertex() -> ref<Vertex>
  FAIL WITH "pool exhausted" IF handed >= reserve - 1
  record <- records[handed]
  handed <- handed + 1
  RETURN record
```

**Notes** — the ceiling is checked against `reserve - 1`, not `reserve`: one slot is left unused.
Nothing in this file reads that last slot, and the source gives no reason for the margin. Treat
it as a defensive one-slot guard rather than a load-bearing requirement.

## `init` / `get_visited_node_count`

**Contract** — `init` resets the counter to zero, which is the entire cost of starting a new
search as far as the pool is concerned. `get_visited_node_count` returns the counter, which the
search's budget check compares against the caller's ceiling on distinct vertices visited.
