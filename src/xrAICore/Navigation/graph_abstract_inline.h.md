# src/xrAICore/Navigation/graph_abstract_inline.h

> The mutable graph's operations — vertex and edge lifecycle with an edge count kept incrementally, and a chunked save/load that stores vertices and adjacency separately.

**Needs** — [`graph_abstract.h`](graph_abstract.h.md) · [`graph_vertex_inline.h`](graph_vertex_inline.h.md) · [`graph_edge_inline.h`](graph_edge_inline.h.md) · [`../../xrCore/FS.h`](../../xrCore/FS.h.md)
**Used by** — [`graph_abstract.h`](graph_abstract.h.md)
**Tier floor** — T2: container manipulation and a chunked serialization.

## Purpose

The operations of the runtime-built graph declared in [`graph_abstract.h`](graph_abstract.h.md): vertex and edge lifecycle, the search-facing view, and the chunked serialized form used by the engine's own tools.

## State

```text
RECORD Graph
  vertices   : map<VertexId, Vertex>   # ordered by identity
  edge_count : int                     # total outgoing edges across all vertices
```

**Invariants** — `edge_count` is maintained incrementally by the vertices, which each hold a
reference to it and adjust it as their own adjacency changes; it is never recomputed. Clearing
the graph asserts it has fallen back to zero, which is the only check that the incremental
bookkeeping is sound. Vertex identities are unique: adding a vertex that already exists is a
programming error, not an update.

## `add_vertex` / `remove_vertex`

**Contract** — `add_vertex` creates a vertex with the given payload under the given identity, and
hands it the shared edge counter. `remove_vertex` destroys the vertex, which as a side effect
removes every edge out of it *and every edge into it* before the vertex disappears. Removing an
absent identity is a programming error.

**Notes** — cleaning up incoming edges is why each vertex keeps a list of the vertices that point
at it (see [`graph_vertex_inline.h`](graph_vertex_inline.h.md)). Without it, removing a vertex
would have to scan every adjacency list in the graph. This is the one piece of redundant state
in the structure and it exists solely to make removal proportional to a vertex's degree rather
than to the graph's size.

## `add_edge` / `remove_edge`

**Contract** — `add_edge` requires both endpoints to exist and adds a directed edge with the given
weight. The two-weight form adds both directions, each with its own weight — the engine's graphs
are directed, and a "bidirectional" edge is two edges that may cost differently. Adding an edge
that already exists is a programming error. `remove_edge` drops the edge from the source's
adjacency and from the destination's incoming list.

## `clear`

**Contract** — remove every vertex, which removes every edge. Implemented by repeatedly removing
the first vertex, so that the incoming-edge bookkeeping runs and the edge counter is verified to
reach zero.

## Equality

**Contract** — two graphs are equal when they have the same vertex count, the same edge count,
and equal vertex sets. Vertex equality compares identity, adjacency and payload. The counts are
checked first because they are cheap and reject almost all mismatches.

## `begin` / `value` / `get_edge_weight` / `is_accessible`

**Contract** — the search-facing view. `begin` hands back the range of a vertex's outgoing edges;
`value` reads the destination identity from an edge; `get_edge_weight` reads its weight;
`is_accessible` is always true for this kind of graph.

**Notes** — `get_edge_weight` takes the two endpoint identities as well as the edge, and uses only
the edge; the identities exist so the signature matches the level mesh's, where the weight must be
computed from both endpoints because the mesh stores no edge weights at all. That shared signature
is what lets one search body walk both.

## `save` / `load` — the serialized form

**Contract** — write the graph to, or read it from, a chunked container in three chunks. Version-
free: this format is internal to the engine's own tools, not one of the frozen game formats.

```text
chunk 0 : vertex count (informational; the loader reads and discards it)
chunk 1 : one sub-chunk per vertex, numbered from zero, each holding
            sub-chunk 0 : the vertex identity
            sub-chunk 1 : the vertex payload
chunk 2 : adjacency, read until the chunk is exhausted:
            source identity
            outgoing edge count (must be non-zero)
            that many times: destination identity, edge weight
          vertices with no outgoing edges are omitted entirely
```

**Invariants** — loading clears the graph first. All vertices must be read before any edge, which
the chunk order guarantees; an edge naming an unknown identity is a corrupt file and fails on the
endpoint assertion. Chunk 2 is optional: a graph with no edges at all saves without it and loads
by stopping early.

**Notes** — the vertex-count chunk is written and never used. Its value is redundant with the
number of sub-chunks in chunk 1, and the loader deliberately ignores it rather than validating
against it. A rebuild may drop the chunk if it does not need to interoperate with files this
version wrote, and must keep writing it if it does.
