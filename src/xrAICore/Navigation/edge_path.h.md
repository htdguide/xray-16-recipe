# src/xrAICore/Navigation/edge_path.h

> Declares the richer path record — remember the edge taken into each vertex as well as the parent, so the answer can be a list of edges.

**Needs** — [`edge_path_inline.h`](edge_path_inline.h.md) · [`vertex_path.h`](vertex_path.h.md)
**Used by** — [`edge_path_inline.h`](edge_path_inline.h.md) · [`graph_engine.h`](graph_engine.h.md)
**Tier floor** — T2: link walking plus one extra field per vertex.

## Purpose

Declares the surface implemented in [`edge_path_inline.h`](edge_path_inline.h.md). Extends the
vertex-only path record with the edge by which each vertex was reached. The planner needs this:
its graph's vertices are world states and its edges are operators, and a plan *is* the list of
operators — the states between them are intermediate bookkeeping the caller never sees.

## State

Stateless. The one per-vertex field it contributes is listed under the exported units below.

## Exported units

- `VertexData` — adds one field to each search vertex: the edge taken into it. What an edge is
  depends on the assembly; for the planner it is an operator identifier.
- `assign_parent(vertex, parent)` — record the parent, leaving the edge field untouched. Used for
  the start vertex, which has neither.
- `assign_parent(vertex, parent, edge)` — record both.
- `get_edge_path(path, best, reverse_order)` — write the sequence of edges from start to goal, or
  goal to start, *appending* to whatever the caller's list already holds.

**Notes** — the edge record sits on the vertex, not on a separate structure, for the same reason
every other search field does: one flat record per vertex, touched many times per expansion.
Both the vertex list and the edge list are available from the same search — the edge policy
inherits the vertex one — so a caller that wants both pays for one search.
