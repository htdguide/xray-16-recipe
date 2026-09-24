# src/xrEngine/xrSheduler.h

> Declares the time-budgeted update scheduler; the substance is in [`xrSheduler.cpp`](xrSheduler.cpp.md).

**Needs** — [`xrSheduler.cpp`](xrSheduler.cpp.md) · [`ISheduled.h`](ISheduled.h.md) · [`xrCore/FTimer.h`](../xrCore/FTimer.h.md) · [`xrCore/xrstring.h`](../xrCore/xrstring.h.md)
**Used by** — [`Engine.cpp`](Engine.cpp.md) · [`Engine.h`](Engine.h.md) · [`ISheduled.cpp`](ISheduled.cpp.md) · [`ISheduled.h`](ISheduled.h.md) · [`xrSheduler.cpp`](xrSheduler.cpp.md) · [`xr_object_list.cpp`](xr_object_list.cpp.md)
**Tier floor** — T1.

## Purpose

Declares the surface described in [`xrSheduler.cpp`](xrSheduler.cpp.md), and one record of
its own: a scheduled item is (due time, last run time, captured name, object), ordered so
that the *earliest due* item compares highest — an inverted comparison, because the
underlying heap is a max-heap and the scheduler wants a min-heap on time.

Exported units:

- **`CSheduler`** — the scheduler. Registration (`Register`, `Unregister`, `Registered`),
  the per-frame drive (`Update`, `Process`, `ProcessStep`), the pairwise ordering hint
  (`EnsureOrder`), lifecycle (`Initialize`, `Destroy`) and the statistics readout.
- The budget state is public on the object — the start and limit in cycle-counter ticks —
  because the dispatch loop and its callers both read it.
