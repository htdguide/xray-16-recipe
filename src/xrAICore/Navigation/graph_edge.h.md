# src/xrAICore/Navigation/graph_edge.h

> Declares an edge of the mutable graph — a weight, a destination, and optionally a payload of its own.

**Needs** — [`graph_edge_inline.h`](graph_edge_inline.h.md)
**Used by** — [`graph_abstract.h`](graph_abstract.h.md) · [`graph_edge_inline.h`](graph_edge_inline.h.md) · [`graph_vertex.h`](graph_vertex.h.md)
**Tier floor** — T2: a pair of fields.

## Purpose

Declares the surface implemented in [`graph_edge_inline.h`](graph_edge_inline.h.md). An edge of
the runtime-built graph: it names where it goes and what it costs.

## State

Stateless. The edge record is in [`graph_edge_inline.h`](graph_edge_inline.h.md).

## Exported units

- `EdgeBase` — the weight and the destination vertex. The destination is held directly rather
  than by identity, so following an edge is a dereference rather than a map lookup; `vertex_id()`
  reads the identity out of the destination.
- `Edge` — `EdgeBase` plus an optional payload, for graphs that need to record something about
  the traversal itself rather than about either endpoint.
- comparison against a destination identity — this is how a vertex finds an edge by scanning its
  adjacency list.
- equality between two edges — equal weight and equal destination identity; a payload, if any,
  is not compared.

**Notes** — the payload-free case is not the payload case with an empty payload: it is a separate
shape carrying no payload field at all, so an edge in a graph that needs no edge data costs
exactly a weight and a reference. For a level compiler holding millions of edges that difference
is the whole memory budget. A rebuild in a language with zero-sized fields gets this for free; one
without needs two shapes, as here.

Edge equality ignores the payload while vertex equality includes the payload. Nothing in the
source explains the asymmetry; treat it as an inconsistency rather than a rule.
