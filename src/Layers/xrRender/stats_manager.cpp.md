# src/Layers/xrRender/stats_manager.cpp

> Keeps a running total of how many bytes of device memory the renderer has allocated, split by what the memory is for and which pool it came from.

**Needs** — [`stats_manager.h`](stats_manager.h.md) · [`HWCaps.h`](HWCaps.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`stats_manager.h`](stats_manager.h.md)
**Tier floor** — T1: it derives an allocation's size from a device object's own description, including a per-pixel size for every surface format.

## Purpose

There is no way to ask a graphics device how much memory a resource costs, so the engine counts for itself. Every vertex buffer, index buffer and render target created by the renderer is registered here and deregistered when released, and the console reports the totals.

The counting is **derived, not recorded**: rather than asking the caller for a size, each entry point reads the object's own description back from the device and computes the size from it. That is deliberate — a size the caller states is the size it *asked for*, which is not what the device allocated.

## State

```text
RECORD MemoryStats
  totals : grid[purpose][pool] of int    # bytes
  # debug builds only:
  live   : list<{ object, size, purpose, pool }>

ENUM Purpose { vertex_buffer, index_buffer, render_target }
ENUM Pool    { device_local, driver_managed, system_memory, scratch }
```

Invariants:

- Both indices are bounds-checked on every operation. The totals grid is fixed-size and an out-of-range purpose or pool is a programming error, not a data condition.
- A decrement must be matched by an earlier increment of the same size. In a debug build the live list enforces this: a decrement that finds no matching entry is an assertion failure, which catches both double-frees and resources that bypassed the registration.
- On a dedicated server every operation returns immediately. There is no device, no resource and nothing to count; the guard is at the top of every entry point rather than at the call sites.

## The reference-count guard

**Contract** — The release side of each entry point does something that looks strange and is essential: it takes a reference on the object, immediately drops it, and **returns without counting if more than one reference remains**.

The reason is that the renderer's resources are shared. A render target may be held by several passes; releasing one holder does not free the memory. Counting the release when the object survives would drive the totals negative. Adding and dropping a reference is how the code reads the current count without a device call.

A rebuild whose resource handles expose their own use count does this directly; the manoeuvre is a workaround for an interface that does not.

## `increment` / `decrement`

**Contract** — Add or subtract a byte count from one cell of the totals grid, and in a debug build register or unregister the object in the live list. The simple form takes a size, purpose and pool; the tracked form also takes the object, and only the tracked form maintains the live list.

## `increment_render_target` / `decrement_render_target`

**Contract** — Derive a texture's size from its description: height times width times the bytes per pixel of its format, filed under the device-local pool. Mip chains and array slices are **not** counted — only the top level of one slice. That undercounts a mipped target by a third; nothing mipped is used as a render target in practice, so the totals stay honest, but a rebuild that mips its targets must fix it.

## `increment_vertex_buffer` / `decrement_vertex_buffer`, and the index-buffer pair

**Contract** — Derive a buffer's size from its description's byte width, filed under the driver-managed pool. Exact — a buffer's byte width is what was allocated.

**Notes** — Filing every buffer under "driver-managed" is a carry-over from an older device model that really had distinct pools; on a modern device every one of these lives in device-local memory. The pool axis is therefore nearly degenerate in practice, and the console report that splits by it prints zeroes in most columns. A rebuild should either drop the axis or map it onto something the target device actually distinguishes.

## `bytes_per_pixel`

**Contract** — The size of one pixel of a given surface format, or zero for a format not listed. Two versions exist — one per format enumeration the two backends use.

**Invariants** — The modern form exploits the fact that the format enumeration is *ordered by width*: every sixteen-byte format precedes every twelve-byte one, and so on, so the function is a handful of range comparisons rather than a table. That is a real dependency on the enumeration's layout, and it is sound because that enumeration is fixed by the graphics API. A rebuild against a different API must write its own mapping and cannot assume the ordering.

Both forms return zero for block-compressed and other "extraordinary" formats rather than guessing. Compressed textures are therefore not counted at all — and they are the largest single consumer of device memory. The totals this file produces are consequently *not* a memory budget; they are a leak detector and a sanity check on the buffers the renderer itself allocates.

## Teardown

In a debug build, teardown reports how many entries the live list still holds. The assertion that it be empty is present but disabled, because the renderer legitimately leaves some resources alive past this point.
