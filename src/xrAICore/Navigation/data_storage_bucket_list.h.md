# src/xrAICore/Navigation/data_storage_bucket_list.h

> Declares the approximate frontier — cost space sliced into a fixed number of buckets, each a sorted list of open vertices.

**Needs** — [`data_storage_bucket_list_inline.h`](data_storage_bucket_list_inline.h.md)
**Used by** — [`data_storage_bucket_list_inline.h`](data_storage_bucket_list_inline.h.md) · [`graph_engine.h`](graph_engine.h.md)
**Tier floor** — T2: an ordering structure; its per-vertex links are an efficiency decision, not a layout requirement.

## Purpose

Declares the surface implemented in
[`data_storage_bucket_list_inline.h`](data_storage_bucket_list_inline.h.md). This is the frontier
the level and game navigation searches use. It trades exactness for speed: costs are mapped onto
a fixed ladder of buckets and the search always takes from the lowest non-empty bucket, so
`get_best` returns a vertex within one bucket-width of the true minimum rather than the true
minimum itself.

## State

Stateless. The frontier's state and the per-vertex fields it needs are in [`data_storage_bucket_list_inline.h`](data_storage_bucket_list_inline.h.md).

## The assembly parameters

- **bucket count** — how finely cost space is sliced. The navigation search uses 8192.
- **generation-stamp types** — the width of the per-vertex path generation and bucket index.
- **clear-buckets** — whether the bucket array is wiped at the start of every search (simple,
  costs a wipe per search) or left stale and validated per entry by generation stamp (no wipe,
  costs a stamp check on every read). The navigation search chooses *not* to wipe.

## Exported units

- `VertexData` — the fields a vertex needs to live in this frontier: forward and backward links
  within its bucket, the search generation it was placed in, and which bucket that was.
- `init` — begin a new search.
- `is_opened_empty` — whether any bucket still holds a live vertex; this call *advances* the
  lowest-bucket cursor as a side effect (see the inline twin).
- `add_opened`, `decrease_opened`, `remove_best_opened`, `add_best_closed`, `get_best` — the
  frontier operations the search calls.
- `compute_bucket_id(vertex)` — map a cost to a bucket.
- `set_min_bucket_value` / `set_max_bucket_value` — set the cost range the ladder spans. The
  caller tunes this to the search: the navigation engine spans zero to two thousand.

**Notes** — the per-vertex links mean a vertex knows where it sits in the frontier, so lowering
a vertex's cost is an unlink and a re-insert rather than a search. That is the whole reason this
structure is preferred for navigation, where cost improvements are frequent and the frontier
holds many thousands of vertices.
