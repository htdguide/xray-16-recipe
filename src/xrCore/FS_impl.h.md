# src/xrCore/FS_impl.h

> The chunk-lookup strategy, isolated so it can be swapped and measured.

**Needs** — [`FS.h`](FS.h.md) · [`FTimer.h`](FTimer.h.md)
**Used by** — [`FS.cpp`](FS.cpp.md) · [`FS.h`](FS.h.md) · [`stream_reader.cpp`](stream_reader.cpp.md)
**Tier floor** — T2: it is a search over a list of (id, offset) pairs; nothing here touches the device or a byte layout.

## Purpose

Holds the body of chunk lookup for every reader flavour, in four interchangeable implementations plus an optional call-and-time counter. Only one is compiled; the others are kept because the choice was made by measurement and a rebuilder deserves to see the alternatives before re-deriving them. The behaviour they must all agree on — what is returned, how the compression mark is handled, what "not found" means — is contracted in [`FS.cpp`](FS.cpp.md) under `find_chunk`.

This is a separate file only because the code must be instantiated per reader flavour; in a rebuild it is a method, and this file disappears.

## `find_chunk` — the four strategies

**Contract (identical for all four)** — return the payload length of the first chunk whose identifier, ignoring the compression mark, equals the requested one, leaving the cursor at the start of that payload; return zero if there is none. Optionally report the mark.

**Invariants** — the strategies must be observationally identical, including *which* chunk wins when an identifier repeats (the first in file order) and the fact that a failed search leaves the reader usable.

```text
# 1. plain  — rewind, walk, skip by stored size. No state.
#    Cost: O(chunks) per lookup, O(chunks^2) for a full read.

# 2. one-probe heuristic  — remember where the last successful chunk ended and
#    try that position first; fall back to the plain walk. One integer of state.
#    Cost: O(1) per lookup when chunks are read in written order, which is the
#    overwhelmingly common case. THIS IS THE ONE THAT SHIPS.

# 3. pre-scanned vector  — on first lookup, walk the whole container twice
#    (once to count, once to fill) into a sorted array of (id -> offset), then
#    binary-search. Pays the whole walk even when one chunk is wanted.

# 4. lazy hash map  — walk forward from wherever the last walk stopped,
#    recording every (id -> offset) seen, and stop when the wanted id appears.
#    Amortizes like 3 but never over-scans. Costs a hash map per reader.
```

**Notes** — strategies 3 and 4 allocate per reader, and readers are created per chunk during a level load, thousands of them. That allocation is what sank them, not the search itself. A rebuilder with a cheap allocator should re-measure rather than assume.

## Call counter

**Contract** — when enabled, counts lookups and accumulates the wall time spent in them across all threads, and can print the total. Purely diagnostic; compiled out by default.
