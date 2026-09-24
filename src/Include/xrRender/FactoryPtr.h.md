# src/Include/xrRender/FactoryPtr.h

> The ownership convention for every factory-made renderer companion: create on construction, destroy on destruction, copy by asking the object to copy itself.

**Needs** — [`RenderFactory.h`](RenderFactory.h.md) · [`xrAPI/xrAPI.h`](../xrAPI/xrAPI.h.md)
**Used by** — [`DebugShader.h`](DebugShader.h.md) · [`WallMarkArray.h`](WallMarkArray.h.md) · [`thunderbolt.h`](../../xrEngine/thunderbolt.h.md) · [`xr_efflensflare.h`](../../xrEngine/xr_efflensflare.h.md) · [`HitMarker.h`](../../xrGame/HitMarker.h.md) · [`wallmark_manager.h`](../../xrGame/wallmark_manager.h.md)
**Tier floor** — T1: the whole file exists to pin a destruction *time* to a lexical scope, across a dynamically loaded module boundary. A tier with non-deterministic finalization must replace it with explicit release, not merely re-express it.

## Purpose

An engine-side object that owns a renderer companion — a font, a weather keyframe, a UI element — should not have to remember to create it in its constructor and free it in its destructor through the right factory method. This file makes that automatic: a field of this type *is* the companion, allocated when the owner is, released when the owner is.

It is a handle type and a policy, not an algorithm. Its interest to a rebuilder is entirely in the policy.

## State

```text
RECORD FactoryHandle<T>
  object : optional<T>       # none only after the handle has been moved out of
```

**Invariants** — a freshly constructed handle always holds an object; construction cannot fail. A handle that has been moved from holds nothing and must not be dereferenced, but may still be destroyed.

## The ownership policy

```text
ON construct         : object = render_factory.create<T>()
ON destroy           : render_factory.destroy<T>(object); object = none
ON copy-construct    : object = render_factory.create<T>(); object.copy_from(source.object)
ON copy-assign       : object.copy_from(source.object)      # the existing object is reused
ON move              : take the source's object; the source is left holding none
```

**Contract** — copying a handle makes a *new companion and asks the companion to duplicate itself*, rather than sharing one. Assignment does not reallocate: it overwrites the existing companion's contents. This is why almost every interface in this directory declares a `copy` method that looks redundant — it is the hook this policy calls, and it is the reason those methods exist at all.

**Notes** — Three decisions here survive translation and one does not.

**Copy means deep copy.** The companions are small and per-owner; sharing one between two owners would make destruction ambiguous across the module boundary. A rebuild is free to reference-count instead, but then must decide which module runs the final release.

**Assignment reuses the object.** The consequence is that a companion's identity is stable across assignment — anything the device holds a pointer to (a texture binding, a buffer) stays valid. Reallocating on assignment would be simpler and would invalidate those.

**Destruction is where the release happens, and the release must cross back into the renderer module.** This is the load-bearing part, and it is why the file exists rather than a generic smart handle being used.

What does not survive: the file contains a compiler-conditional variant in which destruction *leaks* the object instead of freeing it, on the grounds that one compiler's member-destruction order made the release unsafe. That is a workaround for a real bug that was never diagnosed, and it means one supported build configuration leaks every renderer companion for the lifetime of the process. A rebuild must not reproduce it; it must instead establish a defined order — release every companion before the renderer module is torn down — which is the actual missing invariant.

Also incidental: the per-type specializations are written once per companion type by macro, because the create and destroy calls are named methods rather than a generic one. A rebuild with generics writes the policy once.
