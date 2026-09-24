# src/xrCDB/xrXRC.h

> The per-thread query handle — one collider plus the timing counters that make
> query cost visible in the debug overlay.

**Needs** — [`xrCDB.h`](xrCDB.h.md)
**Used by** — [`xrXRC.cpp`](xrXRC.cpp.md) · [`xr_area.h`](xr_area.h.md) · [`stdafx.h`](../xrEngine/stdafx.h.md)
**Tier floor** — T2: it is a small aggregate and three timers; nothing here forces a lower
tier, though it inherits the collider's.

## Purpose

A collision query needs somewhere to put its results, and that somewhere must not be shared
between threads. This type is that somewhere, plus the instrumentation around it, plus the
decision that the two travel together: every query in the engine goes through a handle, so
every query is counted.

It is the substance holder of its pair — [`xrXRC.cpp`](xrXRC.cpp.md) contains only the
process-wide instance and the overlay dump — because the type is entirely inline, which is
itself the decision: a query wrapper that adds a function call per query to a path called
tens of thousands of times a frame is not acceptable, so the wrapper must disappear.

## State

```text
RECORD QueryHandle
  collider : Collider            # holds the result buffer; see xrCDB.cpp
  name     : text                # for the overlay, so two handles can be told apart
  stats    : QueryStats

RECORD QueryStats
  ray_time, box_time, frustum_time : FrameTimer   # per-frame accumulated time and count
  ray_rate, box_rate               : real         # smoothed queries-per-millisecond

# Invariant
#   a handle is owned by exactly one thread and is never passed between threads
```

## `QueryHandle.ray_query` · `box_query` · `frustum_query`

**Contract** — time the call and forward it to the collider, which is where the contracts
actually are ([`xrCDB_ray.cpp`](xrCDB_ray.cpp.md),
[`xrCDB_box.cpp`](xrCDB_box.cpp.md), [`xrCDB_frustum.cpp`](xrCDB_frustum.cpp.md)).

## `QueryHandle.results` · `count` · `clear` · `release`

**Contract** — pure delegation to the collider's result buffer. A caller reads the results
after a query and before the next one; they are not preserved across queries.

## `QueryHandle.dump_statistics`

**Contract** — writes one line per query kind into the debug overlay: total milliseconds
this frame, query count, and the smoothed rate. Rolls the frame over as a side effect, so
calling it twice in a frame loses a frame of numbers. Implemented in
[`xrXRC.cpp`](xrXRC.cpp.md).

## Notes

The rate is smoothed with a heavy first-order filter — each frame contributes one part in a
hundred — because a raw per-frame queries-per-millisecond figure is unreadable, and a
not-a-number from a frame with zero queries is substituted with zero rather than allowed to
poison the running average. That substitution is the only interesting line in the whole
type, and it exists because a division by a zero elapsed time would otherwise permanently
destroy the counter.

**The layering here is inverted and a rebuild should fix it.** The overlay dump takes the
engine's font and performance-alert interfaces, so the collision module — chapter 7 —
depends on a type from chapter 13. The decision under it is only *query cost must be
observable*; a rebuild should have the handle expose its counters and let the overlay read
them.
