# src/xrCore/xrMemory.h

> Declares the allocation routing point implemented in [`xrMemory.cpp`](xrMemory.cpp.md), and the object-lifetime helpers every other file uses instead of the language's own.

**Needs** — [`xrMemory.cpp`](xrMemory.cpp.md) · [`xr_types.h`](xr_types.h.md) · [Seam: Allocator](../../SYSTEM-REQUIREMENTS.md#seam-allocator)
**Used by** — [`object_cloner.h`](../Common/object_cloner.h.md) · [`object_destroyer.h`](../Common/object_destroyer.h.md) · [`xr_allocator.h`](../xrCommon/xr_allocator.h.md) · [`xr_smart_pointers.h`](../xrCommon/xr_smart_pointers.h.md) · [`FS.cpp`](FS.cpp.md) · [`LzHuf.cpp`](LzHuf.cpp.md) · [`Image.cpp`](Media/Image.cpp.md) · [`xalloc.h`](Memory/xalloc.h.md) · [`Lock.cpp`](Threading/Lock.cpp.md) · [`tinystr.cpp`](XML/tinystr.cpp.md) · [`tinystr.h`](XML/tinystr.h.md) · [`tinyxml.cpp`](XML/tinyxml.cpp.md) · [`tinyxml.h`](XML/tinyxml.h.md) · [`_stl_extensions.h`](_stl_extensions.h.md) · _and 13 more_
**Tier floor** — T1: it redefines the language's allocation operators and exposes alignment as a caller-visible parameter.

## Purpose

Declares the allocator described in [`xrMemory.cpp`](xrMemory.cpp.md). Its own substance is the small set of helpers that every other file in the engine reaches for in place of the language's own object creation and destruction — and one decision inside them that a rebuild must reproduce or understand.

## Exported units

- **The allocator object and its global instance** — allocate, aligned allocate, non-throwing allocate, reallocate, free, aligned free, the small-block pair, usage query, compaction, bring-up and teardown.
- **Small-size threshold** — `128 * pointer_size`. Blocks at or below it may use the small path; above it they may not.
- **Scoped small buffer** — picks the small path or the general one by size at construction and releases correspondingly. Movable, not copyable. This is the safe way to use the split and should be the only way.
- **Raw array allocate and free** — typed count in, typed pointer out; free nulls the caller's pointer.
- **Object create and destroy** — allocate, construct in place, and the inverse. Destroy nulls the caller's pointer, including through a const reference, so that a destroyed object cannot be reached again through the same name.
- **Malloc, realloc and string duplication** — for the places that hand buffers to foreign code.
- **Memory fill helpers** — zero, copy, fill, defined here so the platform's own macro spellings are displaced.

## Notes

**Destroying a polymorphic object adjusts the pointer before freeing.** When the type has a virtual table, the object's address as seen through a base-class pointer is not necessarily the address the allocator handed out; the helper recovers the *complete object's* address first, then runs the destructor, then frees that. A rebuild on a tier with a single object identity deletes this entirely; a rebuild on a tier that shares this hazard must reproduce it, because the engine holds and destroys objects through base pointers constantly.

**Destruction nulls the caller's variable.** Not a safety habit — the engine reads those variables after destruction in several teardown paths and depends on finding nothing there.
