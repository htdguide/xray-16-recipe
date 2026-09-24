# src/xrCommon/xr_deque.h

> The engine's double-ended queue, allocating through the engine allocator.

**Needs** — [`xr_allocator.h`](xr_allocator.h.md)
**Used by** — [`_stl_extensions.h`](../xrCore/_stl_extensions.h.md) · [`filetransfer_node.h`](../xrGame/filetransfer_node.h.md)
**Tier floor** — T3: an allocator injection point, nothing more.

## Purpose

Exists to inject an allocator. It names the standard double-ended queue with the engine's
allocation policy substituted, plus the same two declaration shorthands the array header
carries.

## `xr_deque`

**Contract** — a sequence supporting cheap insertion and removal at both ends, with
positional access.

**Invariants**

```text
# Storage is NOT contiguous.
#   This is the only reason to choose it over the growable array, and the reason
#   callers that need a buffer address use the array instead. Appending at either
#   end leaves existing elements at their addresses, which is what the schedulers
#   and message queues that use it rely on.
```

## Declaration shorthands

**Contract** — two macro forms declaring a named queue type with its iterator type. No
meaning; a rebuild deletes them.
