# src/xrAICore/Navigation/data_storage_bucket_list_inline.h

> The navigation frontier — a fixed ladder of cost buckets, each a doubly-linked sorted list threaded through the vertices themselves, with a monotonically rising cursor at the lowest live bucket.

**Needs** — [`data_storage_bucket_list.h`](data_storage_bucket_list.h.md) · [`vertex_manager_fixed_inline.h`](vertex_manager_fixed_inline.h.md)
**Used by** — [`data_storage_bucket_list.h`](data_storage_bucket_list.h.md)
**Tier floor** — T2: a linked structure over pooled records; the pooling is a frame-budget decision, not a layout requirement.

## Purpose

This is the priority queue every navigation path search runs on, and it is the reason a search
over a level's mesh fits inside a frame. Instead of maintaining an exact ordering, it slices the
cost range into a fixed number of buckets, drops each open vertex into the bucket its cost falls
in, and always takes from the lowest non-empty bucket. Within a bucket the list is kept sorted,
so the vertex returned is the cheapest in that bucket — but a vertex in a lower bucket with a
higher cost cannot exist, and one in the *same* bucket is ordered correctly, so the error is
bounded by nothing worse than the ladder being coarse where two vertices land in one slot.

## State

```text
RECORD BucketFrontier
  buckets          : list<ref<Vertex>>   # fixed length; head of each bucket's sorted list
  min_bucket       : int                 # cursor: lowest bucket that may still hold a vertex
  min_bucket_value : real                # cost mapped to bucket 0
  max_bucket_value : real                # cost mapped to the last bucket
  max_distance     : real                # the largest representable cost; sentinel key
  sentinels        : list<Vertex>        # exactly two: a head and a tail guard record
```

and, on every vertex:

```text
RECORD BucketVertexData
  next, prev  : ref<Vertex>   # position within its bucket's sorted list
  path_id     : int           # the search generation this placement belongs to
  bucket_id   : int           # which bucket the vertex was filed under
```

**Invariants** — a bucket's list is sorted by cost, cheapest first. A vertex's `bucket_id` is the
bucket its list membership is recorded in, which after a cost change may briefly differ from
the bucket its *current* cost maps to — `decrease_opened` unlinks using the recorded value and
relinks using the recomputed one, in that order, and getting that order wrong corrupts the list.
`min_bucket` never moves backwards within a search except when a cheaper vertex is inserted, and
is reset to "past the end" at the start of each search, which is how an empty frontier is
represented.

## `compute_bucket_id`

**Contract** — map a cost onto the ladder. Costs at or below the low end clamp to the first
bucket, costs at or above the high end clamp to the last; in between the mapping is linear.

```text
FUNCTION compute_bucket_id(vertex) -> int
  d <- vertex.f
  IF d >= max_bucket_value  RETURN bucket_count - 1
  IF d <= min_bucket_value  RETURN 0
  RETURN int(bucket_count * (d - min_bucket_value) / (max_bucket_value - min_bucket_value))
```

**Notes** — the clamping is what makes the structure safe rather than merely fast: a path search
whose costs exceed the configured range does not corrupt anything, it just loses resolution at
the top, where the ordering matters least because those vertices are expanded last if at all.
The range is set by the caller — the navigation engine spans zero to two thousand over 8192
buckets, roughly a quarter of a metre per bucket for a cost measured in metres.

## `is_opened_empty`

**Contract** — report whether any open vertex remains, *advancing the cursor* as it goes. This is
not a pure query: it is the frontier's lazy garbage collection, and the search calls it once per
iteration, which is exactly the cadence it needs.

```text
FUNCTION is_opened_empty() -> bool
  IF min_bucket == bucket_count          # cursor already past the end
    RETURN true
  IF buckets[min_bucket] IS NOT none AND is_live(buckets[min_bucket])
    RETURN false
  min_bucket <- min_bucket + 1
  WHILE min_bucket < bucket_count AND NOT is_live(buckets[min_bucket])
    min_bucket <- min_bucket + 1
  RETURN min_bucket >= bucket_count

STEP is_live(head)                       # when buckets are wiped per search: just non-none
  RETURN head IS NOT none
     AND head.path_id == current_path_id      # ... otherwise also check the stamps
     AND head.bucket_id == its own index
```

**Notes** — the cursor advances by at most one bucket per call in the fast path, so a search that
never finds anything still walks the ladder linearly rather than rescanning it.

## Stale buckets and the generation stamp

The bucket array is *not* cleared between searches when the frontier is configured that way, and
that is the file's least obvious decision. Clearing 8192 references at the start of every path
search is a measurable cost when a level runs dozens of searches a second. Instead every entry
carries the generation of the search that placed it, and every read checks two things: that the
generation matches the current search, and that the entry's recorded bucket index matches the
slot it was found in. Either mismatch means the reference is a leftover from a previous search
and the bucket is treated as empty. The second check is needed because a vertex record is reused
from the pool across searches and may legitimately carry the current generation while belonging
to a different bucket.

A rebuild may simply wipe the array per search — the structure supports both, selected at
assembly — at the cost of that wipe.

## `add_to_bucket`

**Contract** — insert a vertex into a bucket's sorted list, stamping it with the current
generation and bucket index, and lower the cursor if this bucket is below it. Insertion is by
linear walk from the head, placing the vertex before the first entry whose cost is not smaller.

```text
FUNCTION add_to_bucket(vertex, bucket_id)
  IF bucket_id < min_bucket
    min_bucket <- bucket_id
  vertex.bucket_id <- bucket_id
  vertex.path_id   <- current_path_id
  head <- buckets[bucket_id]
  IF head IS none OR NOT is_live(head)         # empty or stale: the vertex becomes the list
    buckets[bucket_id] <- vertex
    vertex.next <- none ; vertex.prev <- none
    RETURN
  IF head.f >= vertex.f                        # new cheapest in this bucket
    link vertex before head ; buckets[bucket_id] <- vertex
    RETURN
  walk from head.next to the last entry
    IF entry.f >= vertex.f
      link vertex before entry ; RETURN
  link vertex after the last entry
```

**Notes** — lists are walked, not binary-searched, because a bucket holds a handful of vertices
by construction: if it holds many, the range is misconfigured and the whole structure has
degenerated into one sorted list.

## `add_opened`

**Contract** — mark the vertex open in the layer below, compute its bucket from its current cost,
and insert. No allocation.

## `decrease_opened`

**Contract** — the vertex's cost has fallen; move it to its new bucket. Unlink from the list
recorded on the vertex, then insert under the freshly computed bucket. The old key the search
passes is unused here — the vertex's own recorded bucket is the authoritative unlink location.

```text
FUNCTION decrease_opened(vertex, old_key_ignored)
  new_bucket <- compute_bucket_id(vertex)      # from the already-lowered cost
  IF vertex.prev IS NOT none
    vertex.prev.next <- vertex.next
  ELSE
    buckets[vertex.bucket_id] <- vertex.next   # it was the head of its bucket
  IF vertex.next IS NOT none
    vertex.next.prev <- vertex.prev
  add_to_bucket(vertex, new_bucket)
```

**Invariants** — the cost must already have been lowered before this is called, and the recorded
`bucket_id` must still be the pre-change one. A vertex moved to a strictly lower bucket also
drags the cursor down, which is the only way the cursor ever moves backwards.

## `get_best` / `remove_best_opened` / `add_best_closed`

**Contract** — `get_best` returns the head of the lowest live bucket. `add_best_closed` marks
that vertex closed in the layer below, and `remove_best_opened` advances the bucket's head past
it and clears the new head's backward link. As with the heap frontier, the search closes before
it removes, because closing reads the head.

## `init` and construction

**Contract** — construction fixes the ladder (bucket count is a compile-time property of the
assembled search), defaults the range to zero through a thousand, defaults the sentinel cost to
the largest representable value, and clears the bucket array once. `init` starts a new search:
the two sentinel records are reset into a head/tail pair whose tail carries the sentinel cost,
and the cursor is set past the end so the frontier reads as empty until something is inserted.

**Notes** — there is a bucket-list integrity check in the source, compiled out unconditionally.
What it asserts is worth keeping as prose even though it never runs: every entry in bucket *i*
belongs to the current generation, maps to bucket *i*, and its forward and backward links agree
with its neighbours'; walking a bucket forwards and backwards yields the same count. A rebuild
that gets the unlink-then-relink order wrong in `decrease_opened` fails exactly those
assertions, and fails them silently without them.
