# src/xrPhysics/GeometryBits.cpp

> The coarse static-versus-dynamic collision filter, applied before any contact is
> computed.

**Needs** — [`GeometryBits.h`](GeometryBits.h.md) · [`Geometry.h`](Geometry.h.md) · [`PHWorld.h`](PHWorld.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`GeometryBits.h`](GeometryBits.h.md)
**Tier floor** — T2: two bits and a mask.

## Purpose

There are two layers of collision filtering in this chapter and confusing them wastes a
rebuilder's time. The **fine** layer is the class-and-group rule in
[`PHCollideValidator.h`](PHCollideValidator.h.md), which runs per object pair inside the
engine's own broad-phase. The **coarse** layer is here: a two-bit category mask on the shape
itself, consulted by the dynamics library before the pair ever reaches the engine.

Only one distinction is drawn at this layer — *the level's static mesh* versus *everything
else* — and only one thing uses it: shapes that must never touch the world at all.

## State

```text
ENUM geom_category                   # a bit per category, on every shape
  static   = bit 0                   # the level's compiled triangle soup
  dynamic  = bit 1                   # reserved; never assigned
```

## `set_ignore_static`

**Contract** — removes the static category from the set of categories a given shape will
collide against. Applied to the shape's transform wrapper, which is where the library reads
the mask, and irreversible in practice — nothing restores it.

**Notes** — the one shipped user is the camera-collision probe's character-detecting shape
([`ActorCameraCollision.cpp`](ActorCameraCollision.cpp.md)), which exists purely to notice
other characters and must not be stopped by walls. That is the whole reason this layer
exists, which is worth saying plainly: a rebuild with a richer filtering layer can express
the same thing there and delete this file.

The dynamic bit is defined and never set, and initializing an ordinary shape does nothing,
so every ordinary shape carries the library's default mask of all categories. That asymmetry
is not a bug — it means "collide with everything unless told otherwise" — but it does mean
the enumeration overstates what the mechanism actually distinguishes.
