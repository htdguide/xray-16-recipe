# src/xrAICore/Navigation/vertex_manager_hash_fixed_inline.h

> The visited set as a fixed hash table whose entries are handed out from a ring of index records, so an entry's eviction is the only cleanup that ever happens.

**Needs** — [`vertex_manager_hash_fixed.h`](vertex_manager_hash_fixed.h.md) · [`graph_engine_space.h`](graph_engine_space.h.md)
**Used by** — [`vertex_manager_hash_fixed.h`](vertex_manager_hash_fixed.h.md)
**Tier floor** — T1: preallocated table and record array, generation-stamped; nothing is allocated or cleared during a search.

## Purpose

Answers "have I seen this vertex this search, and where is its record" when the vertex identity
is a world state or an interned name rather than an index. Used by the planner's search and by
the string-keyed search.

## State

```text
RECORD HashedVisitedSet
  buckets    : list<ref<IndexRecord>>   # fixed length; head of each chain
  records    : list<IndexRecord>        # fixed length; handed out in order, never freed
  handed     : int                      # how many records this search has taken
  generation : int (wraps)              # current search stamp; zero is never valid

RECORD IndexRecord
  vertex     : ref<Vertex>    # the search vertex this entry stands for
  next, prev : ref<IndexRecord>   # chain within its bucket
  bucket     : int            # which bucket the chain belongs to
  generation : int            # the search that filed it
```

**Invariants** — a chain head is meaningful only when its stamp is the current generation *and*
its recorded bucket matches the slot it was found in; either mismatch means the whole chain is a
leftover and the bucket reads as empty. Both checks are needed because index records are reused
across searches and a reused record may carry the current stamp while belonging elsewhere.

## `hash_index`

**Contract** — map a vertex identity to a bucket by taking its hash modulo the bucket count. The
hash is supplied per identity type: a world state hashes its own condition set; an interned name
hashes by the identity of its interned storage rather than by its characters.

**Notes** — hashing an interned name by storage identity is only correct because interning
guarantees one storage per distinct value. It also means the bucket a name lands in is not stable
across runs, which is fine here — nothing persists this table.

## `is_visited`

**Contract** — walk the identity's chain, comparing identities. Returns false immediately if the
chain head is stale.

```text
FUNCTION is_visited(vertex_id) -> bool
  b <- hash_index(vertex_id)
  entry <- buckets[b]
  IF entry IS none OR entry.generation != generation OR entry.bucket != b
    RETURN false                       # stale chain: treat the bucket as empty
  WHILE entry IS NOT none
    IF entry.vertex.index == vertex_id  RETURN true
    entry <- entry.next
  RETURN false
```

## `get_node`

**Contract** — the search vertex for a visited identity, found by the same chain walk. Calling it
for an unvisited identity is a programming error.

## `create_vertex`

**Contract** — bind a pooled search vertex to an identity and file it. The index record is taken
from the ring in order; because records are reused rather than freed, taking one may *evict* the
entry it previously held, and that eviction must unlink the old chain cleanly before the record
is reused.

```text
FUNCTION create_vertex(vertex, vertex_id) -> ref<Vertex>
  FAIL WITH "index records exhausted" IF handed >= record_count
  entry <- records[handed] ; handed <- handed + 1

  # --- evict whatever this record used to be ---
  IF entry.prev IS NOT none
    entry.prev.next <- entry.next
    IF entry.next IS NOT none  entry.next.prev <- entry.prev
  ELSE
    IF entry.next IS NOT none  entry.next.prev <- none
    IF buckets[entry.bucket] IS NOT none AND buckets[entry.bucket].generation != generation
      buckets[entry.bucket] <- none        # the old chain was stale anyway; drop it

  # --- file the new binding at the head of its chain ---
  entry.vertex     <- vertex
  entry.generation <- generation
  vertex.index     <- vertex_id
  b <- hash_index(vertex_id)
  head <- buckets[b]
  IF head IS none OR head.generation != generation OR head.bucket != b
    head <- none                             # stale chain: start a fresh one
  buckets[b] <- entry
  entry.next <- head
  entry.prev <- none
  entry.bucket <- b
  IF head IS NOT none  head.prev <- entry
```

**Notes** — the eviction half is the subtle part and the reason this is not just "push onto a
chain". A record's previous chain may belong to an earlier search, in which case unlinking it is
pointless but harmless; it may belong to the *current* search, in which case unlinking is
mandatory or the chain is left pointing at a record that is about to mean something else. The
code handles both by unlinking unconditionally and only dropping a bucket head when that head is
provably stale.

Because the record ring is walked in order and never rewound within a search, a search can only
evict entries it created itself if it exceeds the ring size — which is the ceiling the search's
visit budget must respect. Exceeding it is a programming error, not a degradation.

## `init`

**Contract** — advance the generation and reset the record hand-out counter. Only if the
generation wraps to zero are the bucket array and record array actually cleared, and the
generation advanced past zero. This is the same bounded-cost trade the direct lookup makes.

## `is_opened` / `is_closed` / `add_opened` / `add_closed`

**Contract** — the open flag lives on the search vertex. Note the difference from the direct
lookup: here `is_closed` is simply "not open", with no visited check, because the only way to
hold a vertex record at all is to have looked it up.
