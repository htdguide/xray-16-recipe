# src/Layers/xrRender/FHierrarhyVisual.cpp

> A model that is only a list of other models, loaded either as references into the level's model table or as nested streams — and the ownership question that distinguishes the two.

**Needs** — [`FHierrarhyVisual.h`](FHierrarhyVisual.h.md) · [`ModelPool.h`](ModelPool.h.md) · [`xrCore/FMesh.hpp`](../../xrCore/FMesh.hpp.md)
**Used by** — reached through its declarations in [`FHierrarhyVisual.h`](FHierrarhyVisual.h.md); callers name that, not this file.
**Tier floor** — T2: it is list management and a lifetime rule; nothing touches the device.

## Purpose

Multi-material models exist because a mesh may only carry one material per draw. A crate with a metal band and a wooden body is two meshes, and this type is what holds them together as one model. The same mechanism carries genuinely composite objects — a vehicle with its wheels, a weapon with its attachments.

The whole file is one decision: **who destroys the children.**

## `load(name, stream, flags)`

**Contract** — reads the shared model header and then the child list, in one of two forms. Fails hard if neither is present.

```text
FUNCTION load(name, stream, flags)
  read the shared model header

  IF the stream has a CHILDREN-BY-REFERENCE chunk         # a LEVEL model
    read a count and that many model ids
    resolve each against the level's model table
    mark the children as NOT owned

  ELSE IF the stream has a CHILDREN chunk                 # a STANDALONE model
    each numbered sub-chunk is a complete nested model stream
    construct each through the model pool, naming it "<parent name>:<ordinal>"
    mark the children as owned

  ELSE
    FAIL WITH invalid model
```

**Invariants**

- **The two forms differ only in ownership.** A level's models are built once by the level loader into a shared table, and a container that names them by id must not destroy them — several containers may name the same child. A standalone model's children exist only inside it and must be destroyed with it.
- The generated child names — the parent's name without its extension, a colon, and a one-based ordinal — are not decoration. They are the keys the model pool caches by, so two instances of the same composite model share their children's geometry. A rebuild must generate *some* stable per-child key from the parent's; the exact spelling matters only for log readability.
- Ordinals start at one, and the chunk numbering starts at zero. The first child is chunk 0 named `:1`. There is no reason for the offset beyond what reads better in a log.
- Children are destroyed through the **model pool**, not directly, because the pool is what decides whether a child's geometry is still referenced by another instance.

## `copy(source)`

**Contract** — duplicates the source's child list by duplicating each child through the model pool, and marks the result as owning them.

**Invariants** — A duplicate **always owns its children**, even when duplicating a level container that did not. This is correct and slightly surprising: the duplicates are fresh per-instance records that nothing else names, so nothing else can be responsible for them. The children's *geometry* is still shared, because that is what duplicating a model through the pool does.

## `release()`

**Contract** — releases each owned child's shared resources without destroying the child records.

**Notes** — Release and destruction are two separate sweeps over the same list with the same ownership test, and both exist because the renderer tears down in two phases: device resources go first, at device destruction, and the model records go later, at level unload. A rebuild with one teardown needs only one.
