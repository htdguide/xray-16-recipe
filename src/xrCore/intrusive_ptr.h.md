# src/xrCore/intrusive_ptr.h

> A reference-counted handle for objects that carry their own count — the second sharing scheme in the module, distinct from the resource handle and used where the object is not a loadable asset.

**Needs** — [`xrMemory.h`](xrMemory.h.md)
**Used by** — [`xrCore.h`](xrCore.h.md) · [`restriction_space.h`](../xrServerEntities/restriction_space.h.md)
**Tier floor** — T2: a count in the object, decremented on scope exit, destroying at zero.

## Purpose

The engine has two reference-counting schemes. [`xr_resource.h`](xr_resource.h.md) is for loadable assets and carries a name and a registration bit; this one is for everything else — objects that are shared between systems but are not managed by an asset manager. A rebuild should ask whether it needs both; the engine does not obviously.

## State

```text
RECORD Counted                 # the base an object must derive from
  ref_count : int              # not atomic

RECORD Handle of T
  target : optional<T>
```

**Invariants**

- **The count lives in the object**, so a handle is one pointer and an object can hand out a handle to itself.
- **The count is not atomic.** Objects shared across threads through this handle are not safe; the engine relies on convention. This differs from the resource handle, whose count *is* atomic — a distinction a rebuild should resolve one way or the other rather than reproduce.
- **Assignment releases the old target before taking the new one.** That is the reverse of the order [`xr_resource.h`](xr_resource.h.md) uses, and it means **self-assignment through this handle destroys the object.** It is a real hazard in the original; a rebuild must use the acquire-then-release order.
- **The base's destruction path swallows every failure.** Destroying an object whose destructor fails leaks it silently rather than propagating. That is a deliberate choice for a handle that runs during unwinding, and it hides bugs; a rebuild on a tier without unwinding deletes the question.

## Exported units

- **`intrusive_base`** — the count, acquire, release (reporting whether it hit zero), a released-yet test, and the destruction step.
- **Handle** — construct empty, from a raw pointer (taking a reference), by copy (taking one) or by move (stealing); assign from any of those; destroy (releasing); dereference; test for emptiness; compare for equality, inequality and order; swap; and read the raw pointer.

## Notes

The type requires at build time that the object derives from the counting base, which is what makes "the count lives in the object" checkable rather than hoped for.

The move-assignment path in the original only clears the source when the moved-from pointer was non-empty, which is correct but reads as an accident; and the raw-pointer accessor hands out a constant pointer even from a mutable handle, which forces callers that need a mutable one to go through the dereference. Neither is a decision worth carrying.
