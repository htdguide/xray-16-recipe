# src/xrAICore/Navigation/data_storage_binary_heap_inline.h

> The exact frontier — a binary min-heap over references to open vertices, with a cheap guard that keeps the smallest-first insertion pattern from degenerating.

**Needs** — [`data_storage_binary_heap.h`](data_storage_binary_heap.h.md)
**Used by** — [`data_storage_binary_heap.h`](data_storage_binary_heap.h.md)
**Tier floor** — T2: an array-backed heap; the only manual-tier property is that the array is preallocated to the search's ceiling.

## Purpose

Keeps the open set of a search ordered so the cheapest vertex is always at hand. It is the exact
alternative to the bucket frontier: `get_best` really does return the minimum, so a search using
this frontier expands vertices in true cost order. It is chosen where costs have no known range
to bucket over — the planner's operator costs and the string-keyed search.

## State

```text
RECORD HeapFrontier
  slots : list<ref<Vertex>>   # sized to the search's vertex ceiling at construction;
                              # never grows, never reallocates
  head  : index into slots    # first live entry
  tail  : index into slots    # one past the last live entry
```

**Invariants** — `head <= tail` always; the frontier is empty exactly when they are equal. The
live range `[head, tail)` satisfies the min-heap property under the ordering "smaller cost is
better". The heap stores references to vertex records owned by the pool, never copies: the search
mutates a vertex's cost in place and then tells the frontier to re-order, which only works if
both see the same record.

## `add_opened`

**Contract** — mark the vertex open in the layer below, then insert it into the heap. Allocates
nothing; the slot is already there.

```text
FUNCTION add_opened(vertex)
  lower_layer.add_opened(vertex)              # sets the vertex's open flag
  IF slots[head] IS none OR slots[head].f < vertex.f
    slots[tail] <- vertex                     # ordinary append
  ELSE
    slots[tail] <- slots[head]                # the newcomer is the new minimum:
    slots[head] <- vertex                     # seat it at the root directly
  tail <- tail + 1
  sift_up(head, tail)
```

**Notes** — the head/newcomer swap before the sift is a shortcut for the common shape of a
search: vertices are frequently discovered in improving order, so the new entry is very often the
new minimum, and placing it at the root first turns a full sift-up into nothing. It changes
performance, not results; a rebuild may drop it and simply append.

## `get_best` / `remove_best_opened` / `add_best_closed`

**Contract** — `get_best` returns the root, the minimum-cost open vertex, without removing it.
`add_best_closed` marks that same root closed in the layer below while it is still in the heap.
`remove_best_opened` then pops the root, shrinking the live range by one. The search always calls
them in that order — close, then pop — because closing reads the root.

## `decrease_opened`

**Contract** — the vertex's cost has just been lowered by the search; restore the heap property.
The vertex's current position is not known to the caller, so it is found by scanning the live
range, and the entry is then sifted up from there.

```text
FUNCTION decrease_opened(vertex, old_key)
  i <- head
  WHILE slots[i] IS NOT vertex          # linear scan; the entry must be present
    i <- i + 1
  sift_up(head, i + 1)                  # only an upward move is possible: cost only fell
```

**Notes** — the old key is accepted and ignored here; it exists for the bucket frontier, which
does need it. The linear scan is the honest cost of a heap without an index-back-pointer: it is
tolerated because this frontier is only used by the two searches whose frontiers stay small (a
thousand to a few thousand vertices). A rebuild that stores each vertex's heap position on the
vertex itself turns this into a constant-time operation, and nothing else in the engine notices.

## Construction

**Contract** — the slot array is allocated once, sized to the search's vertex ceiling, and zeroed;
`init` merely resets head and tail to the start, which is what makes starting a new search free.
The array is released with the search.
