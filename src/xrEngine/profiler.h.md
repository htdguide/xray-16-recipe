# src/xrEngine/profiler.h

> Declares the hierarchical CPU-timing profiler and the scoped sample it is fed by; the substance is in [`profiler.cpp`](profiler.cpp.md).

**Needs** — [`profiler.cpp`](profiler.cpp.md) · [`profiler_inline.h`](profiler_inline.h.md) · [`Engine.h`](Engine.h.md) · [`defines.h`](defines.h.md) · [`IGameFont.hpp`](IGameFont.hpp.md)
**Used by** — [`profiler.cpp`](profiler.cpp.md) · [`profiler_inline.h`](profiler_inline.h.md) · [`Explosive.cpp`](../xrGame/Explosive.cpp.md) · [`alife_update_manager.cpp`](../xrGame/alife_update_manager.cpp.md) · [`cover_manager.h`](../xrGame/cover_manager.h.md) · [`cover_manager_inline.h`](../xrGame/cover_manager_inline.h.md) · [`enemy_manager.cpp`](../xrGame/enemy_manager.cpp.md) · [`memory_manager.cpp`](../xrGame/memory_manager.cpp.md) · [`movement_manager.cpp`](../xrGame/movement_manager.cpp.md) · [`quadtree.h`](../xrGame/quadtree.h.md)
**Tier floor** — T1: samples are taken with the CPU's cycle counter and the sample record's layout is fixed so that collecting one is a two-field store on a hot path.

## Purpose

Declares the surface described in [`profiler.cpp`](profiler.cpp.md), and decides two things
of its own.

**It is compiled out entirely in shipping builds.** The scoping construct becomes an empty
block, so a measured region costs literally nothing rather than costing a disabled branch.
This is why the construct is a statement bracket rather than a scoped value: only a macro
pair can vanish completely, and a rebuild in a language with zero-cost conditional
compilation should reproduce the property, not the mechanism.

**A sample is a pointer plus a duration, nothing more.** The identifier is borrowed, never
copied, which is what keeps the cost of a sample near the cost of reading the clock. The
consequence a rebuilder must respect: **every identifier passed to the profiler must
outlive the frame** — in practice a literal.

Exported units:

- **`CProfilePortion`** — the scoped sample. Reads the cycle counter on entry, and on exit
  submits (identifier, elapsed) to the profiler. Takes an extra condition so a caller can
  decide per-instance whether to measure at all.
- **`CProfileResultPortion`** — one submitted sample: duration and borrowed identifier.
- **`CProfileStats`** — the accumulated per-identifier row: display name, smoothed time,
  minimum, maximum, total, sample count, call count, and the frame it was last updated.
- **`CProfiler`** — the accumulator and the display; see the implementation twin.
- The process-wide profiler handle and its accessor.
- **`START_PROFILE` / `STOP_PROFILE`** — the statement bracket that opens a measured region.
