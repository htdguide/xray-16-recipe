# src/xrCommon/xr_array.h

> The fixed-size array, plus a dead compatibility shim that exists only to preserve a record's size.

**Needs** — _(none)_
**Used by** — [`_stl_extensions.h`](../xrCore/_stl_extensions.h.md) · [`GameSpy_BrowsersWrapper.cpp`](../xrGameSpy/GameSpy_BrowsersWrapper.cpp.md)
**Tier floor** — T3 for the alias; the shim is T1 by nature — it is a statement about byte layout.

## Purpose

Two unrelated things share this file. The first is a name for the fixed-size array, which
needs no allocator (its storage is inline, which is why this is the one container header
that does not reference the allocation policy). The second is a compatibility shim and is
the only interesting line in chapter 2's alias set.

## `xr_array`

**Contract** — a sequence of a compile-time-known length, stored inline in whatever
contains it. No allocation.

## `xr_array_s`

**Contract** — a fixed-size array padded by one unused 32-bit field.

**Notes** — this is a fossil, and it is preserved deliberately. It replaced an older
engine-local array type that carried a live element count alongside its storage. When that
count was removed, the padding stayed, because some record containing one of these arrays
has a size or a field offset that something else depends on — a serialized layout, or a
hard-coded stride.

**This is the one thing in this chapter that could not be recovered from the source.**
Nothing in the tree uses this type any more: the padding preserves a size that no longer
has a reader. Either the dependent record was deleted and the shim was not, or the
dependency was never written down. A rebuild should delete it, and should treat its
existence as a warning that record sizes elsewhere in the engine are load-bearing in ways
that are not marked.
