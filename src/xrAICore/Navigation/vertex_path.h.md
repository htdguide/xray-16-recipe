# src/xrAICore/Navigation/vertex_path.h

> Declares the cheaper of the two path records — remember only which vertex you came from, and reconstruct the answer as a list of vertices.

**Needs** — [`vertex_path_inline.h`](vertex_path_inline.h.md) · [`xrCommon/xr_vector.h`](../../xrCommon/xr_vector.h.md) · [`xrCore/xr_types.h`](../../xrCore/xr_types.h.md)
**Used by** — [`a_star.h`](a_star.h.md) · [`dijkstra.h`](dijkstra.h.md) · [`dijkstra_inline.h`](dijkstra_inline.h.md) · [`edge_path.h`](edge_path.h.md) · [`graph_engine.h`](graph_engine.h.md) · [`vertex_path_inline.h`](vertex_path_inline.h.md)
**Tier floor** — T2: parent links and a reversal; no device, no layout.

## Purpose

Declares the surface implemented in [`vertex_path_inline.h`](vertex_path_inline.h.md). This is
one of the two things a search can be told to remember about how it reached a vertex. This one
remembers the parent vertex only, which is all a navigation path needs: the answer is the list
of mesh vertices to walk through. The other
([`edge_path.h`](edge_path.h.md)) also remembers which edge was taken, which the planner needs
because its answer is a list of *actions*, and an action is an edge.

## State

Stateless. It contributes no per-vertex fields of its own; it reads the parent link the search maintains.

## Exported units

- `VertexData` — contributes no fields of its own. The parent link it uses is contributed by the
  search itself, because the search needs it to compare costs as well.
- `init` — nothing to reset.
- `assign_parent(vertex, parent)` — record how a vertex was reached. A second form takes an edge
  and ignores it, so that a search written against the edge-recording policy compiles unchanged
  against this one.
- `update_successors(vertex)` — the non-euclidian-heuristic repair path; unimplemented here and a
  hard error if reached. See the inline twin.
- `get_node_path(path, best)` — walk parent links back from the goal and write the vertex
  sequence, start first.
