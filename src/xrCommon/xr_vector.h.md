# src/xrCommon/xr_vector.h

> The engine's growable array: a contiguous sequence whose storage comes from the engine allocator.

**Needs** — [`xr_allocator.h`](xr_allocator.h.md)
**Used by** — [`graph_vertex.h`](../xrAICore/Navigation/graph_vertex.h.md) · [`vertex_allocator_fixed.h`](../xrAICore/Navigation/vertex_allocator_fixed.h.md) · [`vertex_path.h`](../xrAICore/Navigation/vertex_path.h.md) · [`xrCDB.h`](../xrCDB/xrCDB.h.md) · [`xr_stack.h`](xr_stack.h.md) · [`AssociativeVector.hpp`](../xrCore/Containers/AssociativeVector.hpp.md) · [`FixedMap.h`](../xrCore/Containers/FixedMap.h.md) · [`StackTrace.h`](../xrCore/Debug/StackTrace.h.md) · [`_stl_extensions.h`](../xrCore/_stl_extensions.h.md) · [`log.h`](../xrCore/log.h.md) · [`xrCore.h`](../xrCore/xrCore.h.md) · [`xrDebug.h`](../xrCore/xrDebug.h.md) · [`xrPool.h`](../xrCore/xrPool.h.md) · [`xr_ini.h`](../xrCore/xr_ini.h.md) · _and 5 more_
**Tier floor** — T3: nothing here is device- or format-facing. The file exists only because the host language has no way to say "all arrays allocate here".

## Purpose

This file exists to inject an allocator, and for no other reason. It names the standard
growable array with the engine's allocation policy substituted, plus two shorthand forms
for declaring a named array type together with its iterator type in one line — a
convenience that made the declaration of several hundred container typedefs across the
engine fit on one line each.

A rebuild uses its own growable array and deletes this file. What it must carry forward is
the allocator contract in [`xr_allocator.h`](xr_allocator.h.md), and the two properties the
rest of the engine relies on without ever stating them.

## `xr_vector`

**Contract** — a growable, contiguous sequence of a single element type.

**Invariants**

```text
# Storage is contiguous and its address is taken.
#   Vertex, index and constant buffers are filled by handing the graphics device
#   the address of element zero and a byte stride. The collision database builds
#   a triangle array and then indexes into it by offset. A rebuild whose array is
#   not a flat block of memory must introduce an explicit "pack into a buffer"
#   step at every such site.

# Growth invalidates every position into the array.
#   The engine caches indices, never pointers or iterators, across any call that
#   may append. Where a pointer is cached, the array was reserved to its final
#   size first. Both patterns appear and neither is commented.
```

## Declaration shorthands

**Contract** — two macro forms declare a named array type and its iterator type together.
One takes (name, element), the other (element, name, iterator-name), the difference being
only whether the iterator's name is derived or given. They carry no meaning; a rebuild
deletes them.
