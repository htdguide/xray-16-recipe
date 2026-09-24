# src/xrAICore/Navigation/graph_edge_inline.h

> An edge is a weight plus a direct reference to its destination; its identity, for lookup, is its destination's.

**Needs** — [`graph_edge.h`](graph_edge.h.md)
**Used by** — [`graph_abstract_inline.h`](graph_abstract_inline.h.md) · [`graph_edge.h`](graph_edge.h.md) · [`graph_vertex_inline.h`](graph_vertex_inline.h.md)
**Tier floor** — T2: field access.

## Purpose

The whole behaviour of an edge of the runtime-built graph. It is field access and two comparisons; the file is here because the comparisons are what let a vertex find an edge by scanning its adjacency list.

## State

```text
RECORD Edge
  weight      : Weight          # the cost of traversing it; the search sums these
  destination : ref<Vertex>     # held directly, never as an identity
  payload     : Data            # only in the variant that has one
```

**Invariants** — the destination is always present; an edge with no destination cannot be
constructed. `vertex_id()` is derived from the destination, never stored, so an edge cannot
disagree with the vertex it points at.

## `weight` / `vertex` / `vertex_id` / `data`

**Contract** — field reads. The payload, where present, is readable and writable in place.

## Comparison against a destination identity

**Contract** — an edge compares equal to a vertex identity when its destination has that identity.
This is what makes "find the edge to B in A's adjacency list" a plain linear search over the
list, without a projection step.

## Equality between edges

**Contract** — equal weights and equal destination identities. Deliberately compares the
destination's *identity* rather than the destination itself, so two edges in two structurally
identical graphs compare equal even though they point at different vertex objects — which is what
the graph-level comparison needs.
