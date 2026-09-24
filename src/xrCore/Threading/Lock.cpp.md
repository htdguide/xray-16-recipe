# src/xrCore/Threading/Lock.cpp

> The engine's recursive mutex, with a hold counter beside it so that "is anyone in here" can be asked without taking it.

**Needs** — [`Lock.hpp`](Lock.hpp.md) · [`xrMemory.h`](../xrMemory.h.md) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`Lock.hpp`](Lock.hpp.md)
**Tier floor** — T2: a recursive mutex and an atomic counter.

## Purpose

Wraps the platform's recursive mutex and adds one thing the platform does not offer: an atomic count of how many times it is currently held. Several places in the engine want to assert "this is called with the lock held" or "nothing is inside this right now" without acquiring, and the counter is the only way to ask.

## State

```text
RECORD Lock
  impl  : recursive_mutex    # held behind an indirection so the platform type
                             # never appears in the header
  depth : int, atomic        # incremented on every successful acquire,
                             # decremented on every release
```

**Invariant** — `depth` is non-zero exactly when some thread holds the mutex, and equals the number of nested acquisitions. It is *not* per-thread: a recursive acquisition by one thread and an acquisition by another are indistinguishable in the count. Reading it tells you the lock is busy, never who has it.

**Invariant** — the counter is maintained with acquire-release ordering so that a reader observing a non-zero count also observes whatever the holder published before acquiring. A relaxed counter would still report the right number and would not give that guarantee.

## `Enter`, `TryEnter`, `Leave`

**Contract** — `Enter` blocks until the mutex is held and then raises the count. `TryEnter` attempts it without blocking and raises the count only if it succeeded, reporting which. `Leave` releases one level and lowers the count. None allocates. The mutex is recursive: a thread already holding it may enter again and must leave once per entry.

**Notes** — the count is raised *after* acquiring and lowered *after* releasing. Lowering after the release opens a window in which the mutex is free but the count still reads non-zero; nothing in the engine depends on the count being exact at that instant, and closing it would mean lowering before release, which opens the opposite window. Either is fine; a rebuild should pick one and say so.

## Construction, destruction, and moving

**Contract** — a default-constructed lock is free with a zero count. Destruction releases the underlying mutex, which must not be held. Copying is forbidden. **Moving is permitted**, and is the unusual part: it steals the other lock's underlying mutex and its count, and then gives the moved-from object a *fresh* mutex with a zero count rather than leaving it empty.

**Invariants** — a moved-from lock is still usable. That is the entire reason for the fresh-mutex step: containers of objects that own a lock are reallocated as the engine grows them, and the moved-from element must remain safe to destroy and, in some paths, to use.

**Notes** — moving a mutex is only safe while nothing is waiting on it or holding it, and nothing here enforces that. It is a pragmatic accommodation of containers, not a capability to rely on. A rebuild should hold locks behind a stable indirection so that growing a container never moves one, and then forbid the move outright.

## The instrumented build

**Contract** — under a build option, every lock carries a name given at construction, and every acquisition is timed and reported to an installed callback as (name, elapsed cycles). Installing no callback disables the timing at negligible cost.

**Notes** — the measurement brackets the whole acquisition including the blocked time, which is the number one wants: it answers "which mutex is the engine waiting on". This path is not compiled into a shipping build, and a rebuild is free to satisfy it with whatever its profiler offers — see [Seam: Profiler and GPU debugging](../../../SYSTEM-REQUIREMENTS.md#seam-profiler-and-gpu-debugging).

The instrumented variant in the original has drifted out of sync with the shipping one — it references fields that no longer exist and would not compile. Treat the shipping path as the specification and the instrumented one as intent.
