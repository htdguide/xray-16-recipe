# src/xrCore/Containers — two collections the standard ones could not be

Part of chapter 6, [`src/xrCore`](../README.md). Three files, two containers, and one
argument each about why the obvious data structure was wrong.

## What this module is responsible for

[`src/xrCommon`](../../xrCommon/README.md) renames the host language's containers. This
directory holds the two the host language does not have, and each exists because a specific
access pattern in the engine made the general-purpose structure the wrong shape:

- a **map that is built once and read constantly**, small enough that a binary search over
  contiguous memory beats a tree, and iterated in key order often enough that the locality
  matters;
- a **map that is rebuilt from scratch every frame**, where the cost that matters is not
  lookup at all but the allocation and release of a few hundred nodes sixty times a second.

Neither is a general replacement for a map, and the twins say so in the strongest terms
available, because the failure mode in both cases is silent: the sorted array degrades to
quadratic insertion, and the arena map degrades to a linked list.

## Where it sits

It rests on [`src/xrCommon`](../../xrCommon/README.md) for the growable array and the
allocator adapter, and on nothing else. The sorted map is used from roughly a hundred places
in chapter 23 (the game); the arena map is used only by the renderer's draw-sorting graph in
chapter 18.

## The load-bearing ideas

**A sorted array is a map when the map is small and read-heavy.** Lookup is logarithmic
either way, but the array pays no pointer chase and no allocation, and iterating it in key
order is a linear memory walk. It pays on insertion — linear, not logarithmic — which is why
it is correct for a table built at load time and wrong for one that grows during play.

**Never freeing is a strategy, not a leak.** The arena map's `clear` sets its occupancy to
zero and keeps the storage. After a few frames it is as large as the worst frame needs and a
frame's worth of bucketing costs no allocations at all. The price is that its values must be
ones whose stale contents are harmless — and that restriction is enforced nowhere.

**Storing links as addresses into your own storage is a decision with consequences.** The
arena map's nodes point at each other by address, so growing the arena invalidates every
link and the grow path rewrites them all. It is also why the container cannot be moved, only
copied. A rebuild that stores indices instead loses nothing and removes both hazards; this
is the clearest "do it differently" in the directory.

**Two things in these files are dead and should not be reproduced.** The sorted map's lazy
sort hook is empty, and its positional-hint insert has a condition that cannot be satisfied.
Both are documented in the twin so that a rebuild does not reconstruct the behaviour they
imply.

## The twins

| File | Role |
|---|---|
| [`AssociativeVector.hpp`](AssociativeVector.hpp.md) | **The sorted-array map**: binary search over contiguous key/value pairs, overwrite-on-collision, and the bulk-insert strategy switch. Substantive. |
| [`AssociativeVectorComparer.hpp`](AssociativeVectorComparer.hpp.md) | Lifts one key ordering to all four call shapes the sort and the searches need. |
| [`FixedMap.h`](FixedMap.h.md) | **The arena-backed tree**: unbalanced, address-linked, rebuilt each frame, never freed. Substantive. |
