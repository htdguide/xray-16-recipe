# src/xrCore/xrsharedmem.h

> Declares the blob interner and its typed handle, implemented in [`xrsharedmem.cpp`](xrsharedmem.cpp.md).

**Needs** — [`xrsharedmem.cpp`](xrsharedmem.cpp.md) · [`../xrCommon/xr_vector.h`](../xrCommon/xr_vector.h.md) · [`../Common/Noncopyable.hpp`](../Common/Noncopyable.hpp.md)
**Used by** — [`SkeletonMotions.cpp`](Animation/SkeletonMotions.cpp.md) · [`SkeletonMotions.hpp`](Animation/SkeletonMotions.hpp.md) · [`xrCore.h`](xrCore.h.md) · [`xrDebug.cpp`](xrDebug.cpp.md) · [`xrMemory.cpp`](xrMemory.cpp.md) · [`xrsharedmem.cpp`](xrsharedmem.cpp.md)
**Tier floor** — T1: the record's payload must begin at a 16-byte offset and be addressable as a typed array in place.

## Purpose

Declares the shared-blob table described in [`xrsharedmem.cpp`](xrsharedmem.cpp.md), plus the three comparison predicates that define its ordering — and those predicates are the substance, because they encode the table's key discipline.

## Exported units

- **`SharedBlob` record** — reference count, checksum, length, alignment padding, payload. Four-byte packed as a structure; the padding field is what puts the payload at a 16-byte offset.
- **Total ordering** — checksum, then length, then a byte comparison. The complete order, used where a strict weak ordering is required.
- **Search ordering** — checksum, then length, and *no* byte comparison. Used for the binary search, because the caller wants the whole equal-key run, not one element of it. Keeping these two separate is deliberate: searching with the total order would compare payloads on every probe.
- **Exact equality** — checksum, then length, then bytes. What decides a hit.
- **`smem_container`** — the table: intern, sweep unreferenced, dump, report savings. Non-copyable.
- **The global table** — created by the allocator's bring-up, after the string interner.
- **`ref_smem` typed handle** — create from (checksum, element count, typed pointer), copy, assign, release, dereference to the payload as a typed array, index, element count, swap, identity compare, reference-count query.

## Notes

The handle's `create` multiplies the caller's element count by the element size to get the byte length, and divides back to report the count. The element type is not stored, so two handles of different types over one blob will each reinterpret it. A rebuild with type-tagged storage should add the tag; nothing in the engine relies on the reinterpretation.

Ordering comparisons on handles are pointer comparisons, not content comparisons — same decision, same consequences, as the interned string handle in [`xrstring.h`](xrstring.h.md).
