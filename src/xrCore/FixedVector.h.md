# src/xrCore/FixedVector.h

> A sequence with a compile-time capacity and no heap, for the short lists the frame loop produces and drops.

**Needs** — [`xr_types.h`](xr_types.h.md) · [`xrDebug_macros.h`](xrDebug_macros.h.md)
**Used by** — [`_d3d_extensions.h`](../Common/_d3d_extensions.h.md) · [`object_comparer.h`](../Common/object_comparer.h.md) · [`game_graph_space.h`](../xrAICore/Navigation/game_graph_space.h.md) · [`Frustum.cpp`](../xrCDB/Frustum.cpp.md) · [`Frustum.h`](../xrCDB/Frustum.h.md) · [`Bone.hpp`](Animation/Bone.hpp.md) · [`_stl_extensions.h`](_stl_extensions.h.md) · [`xr_object.h`](../xrEngine/xr_object.h.md) · [`hit_immunity_space.h`](../xrGame/hit_immunity_space.h.md)
**Tier floor** — T1: its whole reason to exist is that it lives in the caller's stack frame and allocates nothing.

## Purpose

The frame loop produces thousands of short lists a frame — the bones touched by a query, the triangles a ray crossed, the sectors a frustum reached — and each has a small, known upper bound. This is a sequence that holds its elements inline up to that bound. It is marked deprecated in favour of a fixed-size array plus a separate count, which is the same idea with less surface.

The design decision worth carrying over is the *policy*: overflow is a programming error, not a condition. Pushing past the capacity is checked in a debug build and is undefined otherwise. A rebuilder must either pick capacities that genuinely bound the data or change the policy deliberately — silently growing would move an allocation into the frame loop, which is the one thing this type exists to prevent.

## State

```text
RECORD FixedSequence<T, capacity>
  storage : T[capacity]         # always fully constructed, even past `count`
  count   : int (32-bit)

# invariant: 0 <= count <= capacity, enforced by assertion at every mutation.
# invariant: elements at or past `count` are stale but valid — clearing sets
#   count to zero and does not touch storage.
```

## Exported units

**`push_back` / `pop_back` / `clear` / `resize`** — the obvious sequence operations, each asserting the bound. `clear` and `resize` only move the count; they neither construct nor destroy.

**`erase(index)`** — remove one element by shifting every later element down one. Order-preserving and linear; there is no swap-and-pop form.

**`insert(index, value)`** — shift every element from the index up one and store. Asserts the index is *already occupied*, so appending must go through `push_back`.

**`last()` / `inc()`** — the two-step append: `last` hands back a reference to the slot one past the end (asserting there is room) so the caller can fill it in place, and `inc` then admits it. This exists so a large element can be built directly in the sequence rather than built on the stack and copied. `back()`, by contrast, is the last *admitted* element.

**`assign(pointer, count)`** — replace the contents with a bulk copy, asserting the count is at least one and within capacity.

**`equal(other)`** — element-by-element comparison of two sequences with the same capacity, shortcutting on differing lengths.

## Notes

The count is held as a 32-bit value while the indexing operations take the platform's natural word. That mismatch is incidental to C++ and a rebuild should just pick one.

`assign` rejects a count of zero, which makes "assign nothing" an error rather than a clear. That is almost certainly unintentional, and a rebuild should allow it.
