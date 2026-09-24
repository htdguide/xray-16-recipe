# src/Layers/xrRender/r__sync_point.h

> Declares the frame-pacing fence set.

**Needs** — [`r__sync_point.cpp`](r__sync_point.cpp.md) · [`HWCaps.h`](HWCaps.h.md)
**Used by** — [`r__sync_point.cpp`](r__sync_point.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the type implemented in [`r__sync_point.cpp`](r__sync_point.cpp.md): a rotating set of device fences, one per graphics processor, used by the frame loop to bound how far ahead of the device the processor may run.

Exported units:

- **`SyncPoint`** (`R_sync_point`) — holds the fences and the rotation index; exposes `create`, `destroy`, `wait` (with a poll interval and a timeout, reporting whether the device caught up) and `end` (advance and place the next fence).

**Notes** — The fence array is sized by the *maximum* supported processor count, a compile-time constant, and only the first `processor_count` entries are used. That is a fixed-size-array decision; a rebuild may size it at creation.
