# src/xrCDB/xrXRC.cpp

> Defines the one process-wide query handle and renders the query statistics into
> the debug overlay.

**Needs** — [`xrXRC.h`](xrXRC.h.md) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — reached through its declarations in [`xrXRC.h`](xrXRC.h.md); callers name that, not this file.
**Tier floor** — T3: one shared instance and some formatted text.

## Purpose

Two things that had nowhere else to go: the definition of a globally reachable query handle,
and the overlay dump whose body cannot be inline because it needs the engine's font
interface.

## State

One process-wide `QueryHandle` named for the overlay. It is *shared*, not per-thread, which
contradicts the invariant in [`xrXRC.h`](xrXRC.h.md) — and that is exactly why the engine's
own object-space queries do **not** use it but keep a per-thread handle of their own (see
[`xr_area.cpp`](xr_area.cpp.md)). This instance is for callers that are known to run on the
render thread only, principally the renderer's light-visibility sampling.

A rebuild should delete it. A globally reachable mutable query buffer is a data race waiting
for the first caller who is wrong about which thread they are on, and the per-thread handle
is already the pattern everywhere it matters.

## `dump_statistics`

**Contract** — writes three lines — ray, box, frustum — each with this frame's total time,
query count and smoothed rate, then rolls the frame counters over.
