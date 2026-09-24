# src/xrAICore/Navigation/graph_abstract.h

> Declares the general-purpose mutable directed graph — the one the searches run on when the graph is built at runtime rather than loaded as a memory image.

**Needs** — [`graph_abstract_inline.h`](graph_abstract_inline.h.md) · [`graph_vertex.h`](graph_vertex.h.md) · [`graph_edge.h`](graph_edge.h.md) · [`../../Common/object_broker.h`](../../Common/object_broker.h.md) · [`../../xrCommon/xr_map.h`](../../xrCommon/xr_map.h.md) · [`../../xrCore/FS.h`](../../xrCore/FS.h.md)
**Used by** — [`patrol_path.cpp`](PatrolPath/patrol_path.cpp.md) · [`patrol_path.h`](PatrolPath/patrol_path.h.md) · [`graph_abstract_inline.h`](graph_abstract_inline.h.md) · [`PhraseDialog.cpp`](../../xrGame/PhraseDialog.cpp.md) · [`PhraseDialog.h`](../../xrGame/PhraseDialog.h.md) · [`alife_spawn_registry.cpp`](../../xrGame/alife_spawn_registry.cpp.md) · [`alife_spawn_registry.h`](../../xrGame/alife_spawn_registry.h.md) · [`smart_cover_description.h`](../../xrGame/smart_cover_description.h.md) · [`smart_cover_loophole.h`](../../xrGame/smart_cover_loophole.h.md)
**Tier floor** — T2: a map of vertices with adjacency lists; nothing device- or format-facing except the optional serialization, which is chunked and version-free.

## Purpose

Declares the surface implemented in
[`graph_abstract_inline.h`](graph_abstract_inline.h.md). Two of the engine's three graphs are
loaded as frozen memory images and are immutable — the level mesh and the cross-level game graph.
This is the third kind: a graph built in memory, edited, searched, and optionally saved. Patrol
routes, authored waypoint networks and the level compiler's intermediate graphs are all this.

It satisfies the same query surface the searches demand — `begin`, `value`, `get_edge_weight`,
`is_accessible` — so a search does not know which kind of graph it is walking.

## State

Stateless. The graph's state is in [`graph_abstract_inline.h`](graph_abstract_inline.h.md).

## Exported units

- `Graph` — vertices keyed by identity, each with an adjacency list of outgoing weighted edges.
  The vertex payload, the edge weight type, the identity type and an optional edge payload are
  all choices made at assembly.
- `add_vertex` / `remove_vertex` — removing a vertex also removes every edge into it; see the
  inline twin for how that is made cheap.
- `add_edge` (one direction, or both at once with separate weights per direction) /
  `remove_edge`.
- `vertex(id)` / `edge(from, to)` — lookup, returning nothing when absent.
- `vertex_count` / `edge_count` / `empty` / `clear` / `vertices`.
- `header()` — returns the graph itself; present so that code written against the loaded graphs,
  which separate a header from their vertex array, compiles against this one too.
- `get_edge_weight(from, to, edge)` / `value(from, edge)` / `begin(vertex, first, last)` /
  `is_accessible(id)` — the search-facing surface.
- `SerializableGraph` — the same graph plus `save` and `load` over the chunked container format.

**Notes** — `is_accessible` is unconditionally true here. Blocking is a property of the *level
mesh*, where a restrictor can close a vertex off; a runtime-built graph has no such notion and
answers accordingly. A search does not special-case this, it just never skips a vertex.
