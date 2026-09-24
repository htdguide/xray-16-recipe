# src/xrAICore/Navigation/vertex_path_inline.h

> Reconstruct a path by walking parent links back from the goal, sized in one pass and filled in reverse so the caller gets it start-first.

**Needs** — [`vertex_path.h`](vertex_path.h.md)
**Used by** — [`edge_path_inline.h`](edge_path_inline.h.md) · [`vertex_path.h`](vertex_path.h.md)
**Tier floor** — T2: link walking and a reversal.

## Purpose

Turns the search's parent links into the answer the caller asked for: the sequence of graph
vertices from start to goal.

## State

Stateless. It reads the parent links the search wrote onto each vertex.

## `assign_parent`

**Contract** — record that a vertex was reached from a parent. Nothing else is remembered. A
second form accepts an edge and discards it, so a search that was written to pass edges works
against this policy unchanged — this is the seam that lets the same search loop serve both path
records.

## `get_node_path`

**Contract** — write the vertex sequence from start to goal into the caller's list, replacing its
contents. Two passes: count the chain, size the list once, then fill it from the back. Allocates
only the one resize.

```text
FUNCTION get_node_path(path, goal_vertex)
  n <- 1
  v <- goal_vertex
  WHILE v.back IS NOT none
    v <- v.back
    n <- n + 1
  path.resize(n)
  i <- n - 1
  v <- goal_vertex
  path[i] <- v.index                # goal goes last
  WHILE v.back IS NOT none
    v <- v.back
    i <- i - 1
    path[i] <- v.index
  # path[0] is the start vertex
```

**Notes** — sizing first and filling backwards avoids building the path reversed and then
reversing it, which for a path of a few hundred vertices matters less than it once did. What is
load-bearing is the *order*: callers consume the path start-first, walking forward as the
creature moves. The parent chain terminates at the start vertex, whose parent is none — assigned
by the search's initialization, which is the only place a null parent is ever written.

## `update_successors`

**Contract** — unimplemented: reaching it is a hard error.

**Notes** — this is the repair operation a heuristic search needs when it improves a vertex it
had already closed, which can only happen under a heuristic that is not a true lower bound. Every
path manager the engine ships declares its metric euclidian, so the branch that would call this
is dead, and the file states that by refusing to run rather than by doing something plausible.
A rebuild that wants inadmissible heuristics must implement it: it must propagate the improved
cost to every descendant of the repaired vertex, which means the path record would have to keep
forward links as well as backward ones. That is a real structural cost, and declining to pay it
is the decision recorded here.
