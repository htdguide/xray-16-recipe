# src/xrAICore/Navigation/edge_path_inline.h

> Reconstruct the sequence of edges taken, appending to the caller's list and in either direction.

**Needs** — [`edge_path.h`](edge_path.h.md) · [`vertex_path_inline.h`](vertex_path_inline.h.md)
**Used by** — [`edge_path.h`](edge_path.h.md)
**Tier floor** — T2: link walking.

## Purpose

Turns the search's parent-and-edge links into a list of edges. This is how a plan comes out of
the planner's search: the operators to run, in order.

## State

Stateless. It reads the parent and edge links the search wrote onto each vertex.

## `assign_parent`

**Contract** — two forms. With an edge, record both parent and edge on the vertex. Without,
record only the parent, leaving the edge field as it was — used for the start vertex, which was
reached by no edge and whose edge field is therefore never read.

## `get_edge_path`

**Contract** — append the edges of the path to the caller's list. The path holds one fewer edge
than it holds vertices, because the start vertex was entered by no edge; the list grows by
exactly that count. Direction is the caller's choice: forward order writes start-to-goal, reverse
order writes goal-to-start. Allocates only the one resize.

```text
FUNCTION get_edge_path(path, goal_vertex, reverse_order)
  edges <- 0
  v <- goal_vertex
  WHILE v.back IS NOT none                # count edges, not vertices
    v <- v.back
    edges <- edges + 1
  base <- path.length
  path.resize(base + edges)               # append, do not clear

  v <- goal_vertex
  IF NOT reverse_order
    i <- path.length - 1                  # fill backwards from the very end
    WHILE v.back IS NOT none
      path[i] <- v.edge ; i <- i - 1 ; v <- v.back
  ELSE
    i <- base                             # fill forwards from where we started appending
    WHILE v.back IS NOT none
      path[i] <- v.edge ; i <- i + 1 ; v <- v.back
```

**Notes** — appending rather than replacing is deliberate, and is the one thing that separates
this from the vertex reconstruction, which clears. A caller stitching a multi-segment route —
a cross-level path assembled from several searches — accumulates the segments into one list.
The forward-order fill writes from the *end of the whole list* backwards, which is correct only
because the appended region ends there; a rebuild that gets that offset wrong silently overwrites
the caller's earlier segments.
