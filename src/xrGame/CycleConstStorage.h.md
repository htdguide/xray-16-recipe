# src/xrGame/CycleConstStorage.h

> A fixed-capacity ring of recent values, indexed from the oldest, that never allocates and never reports a size.

**Needs** — _(none)_
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: an array and a modulo

## Purpose

A history buffer of compile-time-fixed length for per-frame samples — the kind of thing
that records the last N positions or the last N times and is read back by relative age.
It has no count and no empty state: the buffer is pre-filled with a caller-supplied value
before first use, so a reader always gets N values and never has to ask how many are
valid. That is the load-bearing decision; it means callers can average or difference the
whole window unconditionally, at the cost of the caller having to choose a fill value that
is harmless as a sample.

## State

```text
RECORD CycleStorage<T, N>          # N fixed when the storage is declared
  slots : list<T>                  # exactly N entries, never resized
  first : int                      # index of the oldest slot; also the next write slot
```

Invariants: `first` is always in `[0, N)`. Logical index `i` maps to physical slot
`(first + i) mod N`, so index 0 is the oldest retained sample and index `N-1` the newest.
No slot is ever unwritten once `fill_in` has run.

## `CycleStorage`

**Contract** — construction leaves `first` at zero and the slots whatever the element type
default-constructs to; a caller is expected to `fill_in` before reading. Never allocates.

## `fill_in`

**Contract** — writes one value into every slot, establishing the "always N valid samples"
invariant.

## `push_back`

**Contract** — overwrites the oldest slot with the new value and advances the origin, so
the pushed value becomes logical index `N-1`. Constant time; the displaced value is lost.

## Indexing

**Contract** — reads or writes logical index `i` counting from the oldest. The index is not
bounds-checked and is taken modulo the capacity, so an out-of-range index silently aliases
another sample rather than faulting.

**Notes** — the whole type is an artifact of wanting a heap-free history in a per-frame
path. A rebuild whose language gives it a fixed-size ring, or which can afford a deque,
should use that and keep only the "pre-filled, no count" convention, because callers rely
on it.
