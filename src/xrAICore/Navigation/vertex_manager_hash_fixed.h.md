# src/xrAICore/Navigation/vertex_manager_hash_fixed.h

> Declares the hashed visited-set lookup, for searches whose vertex identity is not a small dense integer.

**Needs** — [`vertex_manager_hash_fixed_inline.h`](vertex_manager_hash_fixed_inline.h.md)
**Used by** — [`graph_engine.h`](graph_engine.h.md) · [`vertex_manager_hash_fixed_inline.h`](vertex_manager_hash_fixed_inline.h.md)
**Tier floor** — T1: a fixed-size hash table over pooled index records, generation-stamped to avoid per-search clearing.

## Purpose

Declares the surface implemented in
[`vertex_manager_hash_fixed_inline.h`](vertex_manager_hash_fixed_inline.h.md). The direct-table
lookup ([`vertex_manager_fixed.h`](vertex_manager_fixed.h.md)) needs vertex identities that index
an array. Two of the engine's searches do not have that: the planner's vertices are *world
states* and the string search's vertices are interned names. Both are hashed instead.

## State

Stateless. The lookup's state is in [`vertex_manager_hash_fixed_inline.h`](vertex_manager_hash_fixed_inline.h.md).

## The assembly parameters

- **hash bucket count** — 256 for the planner, 128 for the string search.
- **index-record count** — the maximum number of distinct vertices one search may visit: 8192 for
  the planner, 1024 for the string search. This is a second, independent ceiling on top of the
  vertex pool's.

## Exported units

- `VertexData` — the identity the record stands for, and whether it is open.
- `IndexRecord` — a hash-table entry: which search vertex, its chain links, which bucket it hashed
  to, and the search generation it belongs to.
- `init`, `is_visited`, `is_opened`, `is_closed`, `get_node`, `create_vertex`, `add_opened`,
  `add_closed`, `current_path_id` — the same surface the direct lookup offers, so the two are
  interchangeable at assembly.
- `hash_index(vertex_id)` — the identity's bucket.

**Notes** — the two supported identity types each get their own hash: a world state hashes its own
condition set (see the planner's state type), and an interned string hashes by the identity of
its interned storage, not by its characters — two equal strings share one interned record, so
pointer identity is value identity, and hashing it is both correct and free. A rebuild whose
string type is not interned must hash the characters instead, and must then also make sure
equality means the same thing in both places.
