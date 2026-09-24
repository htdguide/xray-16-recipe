# src/xrCore/xr_resource.h

> The reference-counted resource base and its smart pointer: what it means for a texture, a shader, a mesh or a sound to be shared by many users and destroyed by the last one.

**Needs** — [`xrstring.h`](xrstring.h.md) · [`xrMemory.h`](xrMemory.h.md)
**Used by** — [`SH_Atomic.h`](../Layers/xrRender/SH_Atomic.h.md) · [`SH_RT.h`](../Layers/xrRender/SH_RT.h.md) · [`r_constants.h`](../Layers/xrRender/r_constants.h.md) · [`xrCore.h`](xrCore.h.md) · [`Sound.h`](../xrSound/Sound.h.md)
**Tier floor** — T2: an atomic counter and a handle that decrements on scope exit. It is T2 rather than T3 because destruction must happen *at the moment the last handle drops* — a graphics resource released a frame late is a device-object leak — which a garbage-collected tier has to express explicitly.

## Purpose

Every loaded asset in the engine is shared: a texture is referenced by many materials, a material by many models, a model by many objects. This file defines the contract for that sharing, and the three-layer base hierarchy is the whole content — each layer adds exactly one thing a resource manager needs.

## State

```text
RECORD Resource                   # layer 1: sharing
  ref_count : int (32-bit, atomic)   # invariant: number of live handles; zero means dead

RECORD FlaggedResource EXTENDS Resource   # layer 2: registration
  flags : int (32-bit)               # bit 0: this resource is registered with its manager

RECORD NamedResource EXTENDS FlaggedResource   # layer 3: identity
  name : InternedString              # the key the manager finds it by
```

**Invariants**

- **The count reaching zero destroys the object immediately**, from inside the handle that decremented it. There is no deferred sweep — unlike the string and blob interners, which defer deliberately. The difference is that a resource owns device objects and the device wants them back at a defined time.
- **The registration bit is the manager's, not the resource's.** A resource that is registered is reachable from its manager's table by name, and the manager must unregister it before the last handle drops or the table holds a dangling entry. The bit exists so the resource itself can assert this on its way out.
- **The name is the manager's key**, interned so lookup is a pointer compare. Setting it returns the interned text so a caller can keep a raw pointer to a name it knows outlives it.
- **Copying a resource does not copy its identity as a shared object.** Copy-assignment *exchanges* the counters rather than copying one over the other, so neither object ends up claiming the other's users. This is a strange operation and exists only because a few resources are value-copied during construction; a rebuild should make resources non-copyable and delete the question.

## `resptr` — the handle

**Contract** — An owning pointer: construction from a raw pointer optionally takes a reference (some call sites hand over a reference they already hold), copy takes one, assignment takes the new one *before* releasing the old (so self-assignment is safe), and destruction releases. Dereferencing does not check for emptiness — a handle is either known non-empty or tested first.

```text
FUNCTION acquire(target) -> void
  IF target exists THEN target.ref_count = target.ref_count + 1
  ATOMICALLY

FUNCTION release() -> void
  IF held is none THEN RETURN
  ATOMICALLY held.ref_count = held.ref_count - 1
  IF held.ref_count == 0 THEN destroy(held)   # immediate, in-place

FUNCTION assign(new_target) -> void
  acquire(new_target)     # first, so assigning a handle to itself is safe
  release()               # then
  held = new_target
```

**Invariants** — The acquire-before-release order is the whole correctness argument for assignment. Reversing it destroys the target when a handle is assigned from another handle on the same object.

**Notes** — The handle's truth test is spelled as a conversion to an opaque member-pointer type rather than to a boolean. That is a dialect workaround for accidental integer conversions and has no meaning in a rebuild: it is simply "a handle tests true when it holds something".

The split between the handle's *storage* layer and its *behaviour* layer, with the storage passed in as a parameter, exists so that a second storage policy — one that does not touch the count — can be substituted. The engine ships one policy; a rebuild may collapse the two.

## Casting and container support

**Contract** — Static and dynamic downcasts that produce a handle of the derived type, ordering by pointer for use as a container key, and a swap. Ordering is pointer ordering and is therefore arbitrary but consistent within a run — the same caveat as every other handle in this module.
