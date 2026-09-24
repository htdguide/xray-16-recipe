# src/xrAICore/Navigation/graph_vertex.h

> Declares a vertex of the mutable graph — a payload, an adjacency list, and the back-references that make removal cheap.

**Needs** — [`graph_vertex_inline.h`](graph_vertex_inline.h.md) · [`graph_edge.h`](graph_edge.h.md) · [`../../Common/object_broker.h`](../../Common/object_broker.h.md) · [`../../xrCommon/xr_vector.h`](../../xrCommon/xr_vector.h.md)
**Used by** — [`graph_abstract.h`](graph_abstract.h.md) · [`graph_vertex_inline.h`](graph_vertex_inline.h.md)
**Tier floor** — T2: two lists and a payload.

## Purpose

Declares the surface implemented in
[`graph_vertex_inline.h`](graph_vertex_inline.h.md). A vertex of the runtime-built graph
described in [`graph_abstract.h`](graph_abstract.h.md).

## State

Stateless. The vertex record is in [`graph_vertex_inline.h`](graph_vertex_inline.h.md).

## Exported units

- `Vertex(payload, id, shared_edge_counter)` — constructed with a reference to the graph's edge
  counter, which it adjusts as its own adjacency changes.
- `add_edge(destination, weight)` / `remove_edge(destination_id)`.
- `on_edge_addition(source)` / `on_edge_removal(source)` — the other half of the back-reference
  bookkeeping, called by the vertex at the other end of an edge.
- `vertex_id()` / `data()` (read and write) / `edges()` / `edge(destination_id)`.
- equality — identity, adjacency and payload all equal.

**Notes** — a vertex holds a reference into the graph that owns it, which makes a vertex
meaningless outside its graph. In a rebuild that is a lifetime coupling to be made explicit:
vertices are not values, they are parts of one structure, and the graph must outlive them.
