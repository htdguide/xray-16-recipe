# src/xrAICore/Navigation/vertex_manager_fixed_inline.h

> The visited set as a full-size table plus a generation counter, so starting a search costs one increment instead of clearing hundreds of thousands of entries.

**Needs** — [`vertex_manager_fixed.h`](vertex_manager_fixed.h.md)
**Used by** — [`data_storage_bucket_list_inline.h`](data_storage_bucket_list_inline.h.md) · [`vertex_manager_fixed.h`](vertex_manager_fixed.h.md)
**Tier floor** — T1: the table is sized to the whole graph and the generation trick exists purely to keep per-search cost off the frame budget.

## Purpose

Answers "have I seen this graph vertex in *this* search, and where is its record" in constant
time and without touching memory proportional to the graph at the start of each search. That
second property is the point: a level's navigation mesh has hundreds of thousands of vertices
and the game runs many searches per second, so a per-search wipe of the table would dominate.

## State

```text
RECORD DirectVisitedSet
  entries      : list<{ generation : int, record : ref<Vertex> }>
                 # one per graph vertex; allocated once, zeroed once
  generation   : int (wraps)       # the current search's stamp; zero is never a valid stamp
  vertex_count : int               # length of entries; also the validity bound on identities
```

and, on every search vertex:

```text
RECORD DirectVertexData
  index  : VertexId   # the graph vertex this record stands for
  opened : bool       # in the frontier (true) or closed (false)
```

**Invariants** — an entry is meaningful exactly when its stamp equals the current generation;
otherwise it is a leftover from an earlier search and the vertex counts as unvisited. A vertex
is *visited* when its entry is stamped; it is then either open or closed, never neither. Vertex
identities are always bounds-checked against the table length before use.

## The generation counter and its wrap

**Contract** — `init` advances the generation by one. If the counter wraps to zero, the whole
table is cleared and the counter advanced again, because a zero stamp would collide with the
zeroed initial state of a never-written entry.

```text
FUNCTION init()
  lower_layers.init()                  # path record, then vertex pool
  generation <- generation + 1
  IF generation == 0                   # wrapped
    clear every entry
    generation <- 1
```

**Notes** — this is the file's one real decision, and it is a bounded-cost trade: every search
is free to start, except one search in every 2^n, which pays a full table clear. The counter's
width sets how often that happens; the navigation search uses a 32-bit counter, so in practice
never. A narrower counter is legal and just makes the rare clear less rare — but it must never
be so narrow that a stale entry from *n* searches ago could carry the current stamp while the
same graph vertex is examined, which would make a fresh vertex read as already visited and
silently break the search.

## `is_visited` / `get_node`

**Contract** — `is_visited` compares the entry's stamp with the current generation. `get_node`
returns the record the entry points at; calling it for an unvisited identity is a programming
error, checked by assertion, not a runtime condition.

## `create_vertex`

**Contract** — bind a record freshly taken from the pool to a graph identity: point the table
entry at it, stamp the entry with the current generation, and record the identity on the record
itself so the path reconstruction can read it back.

```text
FUNCTION create_vertex(record, vertex_id) -> ref<Vertex>
  entries[vertex_id].record     <- record
  entries[vertex_id].generation <- generation
  record.index <- vertex_id
  RETURN record
```

## `is_opened` / `is_closed` / `add_opened` / `add_closed`

**Contract** — the open flag lives on the record, so these are field reads and writes. `is_closed`
is "visited and not open" — a vertex that was never visited is neither open nor closed.

**Notes** — the source carries a commented-out packing of the identity and the open flag into one
bitfield. It is not in force: the fields are separate, and the packing parameter the assembly
takes for it is unused. A rebuild should ignore it.
