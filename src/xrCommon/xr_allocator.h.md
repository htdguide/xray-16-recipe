# src/xrCommon/xr_allocator.h

> Names the one allocation policy every container in the engine is required to use.

**Needs** — [`xrCore/Memory/xalloc.h`](../xrCore/Memory/xalloc.h.md) · [`xrCore/xrMemory.h`](../xrCore/xrMemory.h.md) · [Seam: Allocator](../../SYSTEM-REQUIREMENTS.md#seam-allocator)
**Used by** — [`xr_deque.h`](xr_deque.h.md) · [`xr_list.h`](xr_list.h.md) · [`xr_map.h`](xr_map.h.md) · [`xr_set.h`](xr_set.h.md) · [`xr_string.h`](xr_string.h.md) · [`xr_unordered_map.h`](xr_unordered_map.h.md) · [`xr_vector.h`](xr_vector.h.md) · [`FixedMap.h`](../xrCore/Containers/FixedMap.h.md)
**Tier floor** — T1: the policy exists because the frame loop cannot tolerate an unpredictable collector; a tier that owns its heap deletes this file and inherits the constraint as a pooling problem instead.

## Purpose

This file is one line of source and the whole reason chapter 2 exists. It gives a single
name to the engine's allocation policy so that every container alias in this directory can
spell it in its default parameter. A rebuild in a language whose containers already allocate
from a managed heap will delete this file and every alias that references it — but must
first decide, consciously, whether that heap meets the contract below, because the rest of
the engine was written assuming it.

The substance is the *contract*, not the alias. The filling is described in
[`xrCore/xrMemory.h`](../xrCore/xrMemory.h.md) and is selectable at build time between a
third-party pooled allocator, the system allocator, and a debug pass-through.

## The allocation contract

**Contract** — every container, every object, and every raw buffer in the engine obtains
memory from one process-wide allocator and returns it there. This is enforced three ways
at once, which is belt-and-braces and worth knowing about before a rebuilder is surprised
by the redundancy:

1. Every container alias in this directory names this policy as its default allocator.
2. The global object-construction and destruction operators are replaced process-wide, so
   even code that never mentions the policy routes through it.
3. Raw byte allocation goes through named helpers rather than the platform's own.

A rebuild needs only *one* of those three, and should pick whichever its language makes
natural. What it may not do is let some allocations come from a second heap: the engine
reports its own memory usage, compacts on demand, and the profiler attributes every block,
all of which assume one heap.

**Invariants**

```text
# The allocator is a policy, not an object.
#   Two allocator values are always interchangeable: a buffer allocated through
#   one may be freed through any other. Containers therefore move and swap their
#   storage in constant time, never element-by-element.
#   A rebuild that makes the allocator stateful (an arena per subsystem, say)
#   silently turns every container move into a copy, and every cross-subsystem
#   container swap into a corruption.

# Alignment is at least 16 bytes for every block.
#   The math layer stores 4-wide float values in ordinary records and loads them
#   with aligned instructions; a container of such records must be aligned or the
#   loads fault. See platform assumptions, SSE2 baseline.

# Free is unsized and unaligned at the call site.
#   The caller hands back only the address. The allocator recovers the block size
#   itself. A rebuild whose allocator demands the original size back must thread
#   that size through every container and every raw buffer - a large, mechanical,
#   and entirely avoidable cost.

# A size query exists.
#   Given a block address the allocator can report how large that block is. The
#   memory-statistics display and the debug leak dump both depend on it.
```

**Notes**

*Exhaustion has no policy here.* Allocation returns nothing on failure and nothing in this
layer checks. The engine simply dies on a null dereference somewhere downstream. This is a
real gap, not a subtlety being preserved: a rebuild should decide what running out of memory
means — a level's geometry is requested as one contiguous block of a few hundred megabytes,
and that is exactly the request most likely to fail.

*The count-times-size product is unchecked.* Raw array allocation multiplies element count
by element size with no overflow guard; the maximum-count query that would bound it is
advisory and containers are not obliged to consult it. At 64 bits this is theoretical; at
32 bits, which this engine still builds for, it is reachable with a corrupt archive
directory supplying an element count.

*Why this policy exists at all.* Originally, to serve a 32-bit address space with a
small-block pool that fought fragmentation across a level load. At 64 bits the pooled
allocator matters far less, which is why it is now a build option rather than the only path.
The requirement that survives is the *predictability*, not the pooling: the frame must not
stall in the allocator.
