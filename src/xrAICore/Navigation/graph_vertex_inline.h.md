# src/xrAICore/Navigation/graph_vertex_inline.h

> A vertex maintains its outgoing edges, and separately the list of vertices pointing at it, so that deleting it costs its degree rather than the whole graph.

**Needs** — [`graph_vertex.h`](graph_vertex.h.md) · [`graph_edge_inline.h`](graph_edge_inline.h.md)
**Used by** — [`graph_abstract_inline.h`](graph_abstract_inline.h.md) · [`graph_vertex.h`](graph_vertex.h.md)
**Tier floor** — T2: list manipulation.

## Purpose

The operations of a vertex of the runtime-built graph, and the one structural decision in it: a vertex knows who points at it, so that deleting it costs its degree rather than a scan of the whole graph.

## State

```text
RECORD Vertex
  id            : VertexId
  payload       : Data
  edges         : list<Edge>          # outgoing; each names a destination and a weight
  incoming_from : list<ref<Vertex>>   # vertices that hold an edge to this one
  edge_counter  : ref<int>            # the owning graph's total edge count
```

**Invariants** — the two lists are mirrors: for every outgoing edge from A to B, B's
`incoming_from` holds A exactly once. Both are kept in step by every mutation, and the graph's
edge counter moves with `edges` only. Adding an edge to a destination that is already in the
adjacency is a programming error; so is removing one that is not.

## `add_edge`

**Contract** — append an outgoing edge to a destination with a weight, tell the destination to
record the back-reference, and increment the graph's edge counter.

```text
FUNCTION add_edge(destination, weight)
  FAIL WITH "duplicate edge" IF destination.id IS IN edges
  destination.on_edge_addition(this)
  edges.append(Edge(weight, destination))
  edge_counter <- edge_counter + 1
```

## `remove_edge`

**Contract** — find the outgoing edge by destination identity, tell the destination to drop the
back-reference, erase the edge, and decrement the counter.

## Destruction — the reason the back-references exist

**Contract** — a vertex being destroyed first removes all of its own outgoing edges, then asks
every vertex in its incoming list to remove its edge to this one, and only then releases its
payload. When it is finished no edge anywhere in the graph refers to it.

```text
FUNCTION destroy()
  WHILE edges NOT empty
    remove_edge(edges.last.destination_id)      # drops back-references at the far end
  WHILE incoming_from NOT empty
    incoming_from.last.remove_edge(this.id)     # each removal shortens incoming_from
  release payload
```

**Invariants** — each loop shortens the list it iterates, because the removal at the far end
calls back into this vertex's own bookkeeping. That mutual recursion is what terminates both
loops; a rebuild that snapshots either list first and then iterates the snapshot will act on
entries that have already been removed.

**Notes** — the payload release is wrapped so that a failure to release does not abort the
destruction, leaving a half-unlinked vertex in the graph. That is a defensive measure against
payload types whose teardown can fail, and a rebuild whose payloads cannot fail may drop it.

## `edge(destination_id)` / lookup

**Contract** — linear scan of the adjacency list, returning nothing when absent. Adjacency lists
in this graph are short — a waypoint has a handful of neighbours — so nothing is indexed.

## Equality

**Contract** — same identity, equal adjacency lists (in order), equal payload. Order-sensitivity
of the adjacency comparison is a real constraint: two graphs built by adding the same edges in
different orders compare unequal. The only user of this comparison is the level compiler checking
its own output against a reference, where the build order is deterministic.

## `data()` / `vertex_id()` / `edges()`

**Contract** — field access. The payload is readable and writable in place; the identity is not —
a vertex's identity is fixed at construction because the graph keys on it.
