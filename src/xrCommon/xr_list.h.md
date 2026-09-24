# src/xrCommon/xr_list.h

> The engine's linked sequence, allocating through the engine allocator.

**Needs** — [`xr_allocator.h`](xr_allocator.h.md)
**Used by** — [`_sphere.cpp`](../xrCore/_sphere.cpp.md) · [`_stl_extensions.h`](../xrCore/_stl_extensions.h.md)
**Tier floor** — T3: an allocator injection point, nothing more.

## Purpose

Exists to inject an allocator. It names the standard doubly-linked sequence with the
engine's allocation policy substituted, plus the same two declaration shorthands.

## `xr_list`

**Contract** — a sequence with constant-time insertion and removal anywhere, and no
positional access.

**Invariants**

```text
# A position into the list stays valid across insertion, and across removal of
# any other element.
#   Callers hold positions across mutation - the object registry and several
#   per-frame worklists remove entries while iterating. A rebuild that offers only
#   an index-addressed sequence must restructure those loops, not just swap the
#   type.
```

## Declaration shorthands

**Contract** — two macro forms declaring a named list type with its iterator type. No
meaning; a rebuild deletes them.
