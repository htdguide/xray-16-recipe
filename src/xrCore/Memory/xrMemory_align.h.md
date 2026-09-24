# src/xrCore/Memory/xrMemory_align.h

> Declares aligned allocation, reallocation, release and size query.

**Needs** — [`xrMemory_align.cpp`](xrMemory_align.cpp.md)
**Used by** — [`xrMemory_align.cpp`](xrMemory_align.cpp.md) · [`xrMemory.cpp`](../xrMemory.cpp.md)
**Tier floor** — T1: these are addresses with arithmetic constraints.

## Purpose

Declares the surface implemented in [`xrMemory_align.cpp`](xrMemory_align.cpp.md). Six entry points, each a variant of "get memory whose address satisfies a constraint": allocate aligned, allocate aligned at an offset, the two matching reallocations, release, and query the usable size of a block.

## Exported units

- **`aligned_malloc`** — a block of at least the requested size whose address is a multiple of the requested alignment.
- **`aligned_offset_malloc`** — the same, except the constraint applies to the address *offset bytes into* the block rather than to the block's start.
- **`aligned_realloc`**, **`aligned_offset_realloc`** — resize a block obtained from the matching allocator, preserving its contents up to the smaller of the two sizes and its alignment.
- **`aligned_free`** — release a block from any of the four.
- **`aligned_msize`** — the usable size of the underlying allocation, which is at least the requested size and usually more.
