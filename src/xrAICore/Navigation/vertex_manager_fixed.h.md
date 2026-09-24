# src/xrAICore/Navigation/vertex_manager_fixed.h

> Declares the direct visited-set lookup — one table slot per graph vertex, valid only for the current search generation.

**Needs** — [`vertex_manager_fixed_inline.h`](vertex_manager_fixed_inline.h.md)
**Used by** — [`data_storage_constructor.h`](data_storage_constructor.h.md) · [`graph_engine.h`](graph_engine.h.md) · [`vertex_manager_fixed_inline.h`](vertex_manager_fixed_inline.h.md)
**Tier floor** — T1: a table sized to the whole graph, allocated once; the generation stamping exists to avoid clearing it per search.

## Purpose

Declares the surface implemented in
[`vertex_manager_fixed_inline.h`](vertex_manager_fixed_inline.h.md). A search must answer, many
times per expansion, "have I seen this vertex before, and if so where is its record?". This is
the direct answer: an array indexed by the graph's own vertex identity. It is the lookup the
navigation searches use, because their vertex identities are small dense integers — a level's
navigation mesh numbers its vertices from zero, so the identity *is* the index.

## State

Stateless. The lookup's state and the per-vertex fields it needs are in [`vertex_manager_fixed_inline.h`](vertex_manager_fixed_inline.h.md).

## Exported units

- `VertexData` — the fields this lookup needs on each search vertex: the graph identity it stands
  for, and whether it is currently open.
- `init` — begin a new search by advancing the generation.
- `is_visited(vertex_id)` — has this graph vertex been given a record this search.
- `is_opened(vertex)` / `is_closed(vertex)` — which set a known vertex is in.
- `get_node(vertex_id)` — the record for a visited graph vertex.
- `create_vertex(record, vertex_id)` — bind a freshly pooled record to a graph identity and stamp
  it with the current generation.
- `add_opened` / `add_closed` — flip the open flag.
- `current_path_id` — the current search generation, which the bucket frontier also reads.

**Notes** — the table is sized to the graph's full vertex count at construction, which for a
level's navigation mesh is hundreds of thousands of entries. That is the price of constant-time
lookup with no hashing, and it is paid once per level rather than once per search. A rebuild
targeting graphs whose identities are not dense small integers wants the hash variant instead
(see [`vertex_manager_hash_fixed.h`](vertex_manager_hash_fixed.h.md)).
