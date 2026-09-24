# src/xrEngine/IRenderable.cpp

> Construction, destruction and lazy lighting-cache creation for anything drawable.

**Needs** — [`IRenderable.h`](IRenderable.h.md) · [`Render.h`](Render.h.md) · [`xrCDB/ISpatial.h`](../xrCDB/ISpatial.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`IRenderable.h`](IRenderable.h.md)
**Tier floor** — T1: releases renderer-owned handles at a defined point relative to the frame

## Purpose

Three decisions about the renderable facet's lifetime, and nothing else: how a new
renderable announces itself to the visibility structure, what must be released when it
dies and when that is safe, and when its per-object lighting cache comes into existence.

## State

`Stateless.` The data lives in the `RenderData` record declared in the header.

## Construction

**Contract** — a fresh renderable starts at identity transform with no visual, no lighting
cache, caching permitted, visible, and not in the first-person overlay. If the same object
also carries the *spatial* facet, construction marks it as renderable in the spatial
record, which is what makes the visibility traversal consider it at all. Nothing is
allocated and nothing touches the graphics device.

```text
FUNCTION on_create(self)
  self.render.xform = identity
  self.render.visual = none
  self.render.ros = none
  self.render.ros_allowed = true
  self.render.invisible = false
  self.render.hud = false
  IF self also carries the spatial facet
    self.spatial.type = self.spatial.type WITH flag RENDERABLE
```

**Notes** — the original discovers the spatial facet by a runtime downward type test on a
partially constructed object, which is why it cannot devirtualize this class. That is a
C++ artifact of building one object out of several independent bases. A rebuild composes
the facets explicitly and sets the flag where the object is assembled, paying nothing.

## Destruction

**Contract** — releases the visual and the lighting cache back to the renderer, in that
order, and clears both handles. Both are renderer-owned: the object holds handles, never
storage, so "destroy" means "hand back", and the renderer may keep the underlying asset
alive for other users.

```text
FUNCTION on_destroy(self)
  ASSERT NOT rendering_in_progress      # see note
  renderer.model_delete(self.render.visual)
  IF self.render.ros IS NOT none
    renderer.ros_destroy(self.render.ros)
  self.render.visual = none
  self.render.ros = none
```

**Invariants** — a renderable may not be destroyed while a frame is being recorded. The
engine asserts this against a process-wide "rendering in progress" flag rather than
locking, because the real rule is a *phase* rule: object destruction happens during the
update phase, recording happens during the render phase, and they never overlap. A rebuild
that parallelizes those two phases must replace the assert with a deferred-destruction
queue, not with a mutex.

## `renderable_ros`

**Contract** — returns the per-object lighting cache, creating it on first request. Returns
nothing when caching is forbidden for this object. The cache holds accumulated hemisphere
and sun luminance for the object's position; it is expensive enough that objects which
never need it (particle systems, which are unlit) forbid it outright rather than paying
the creation.

```text
FUNCTION renderable_ros(self) -> optional<ObjectSpecific>
  IF self.render.ros IS none AND self.render.ros_allowed
    self.render.ros = renderer.ros_create(self)
  RETURN self.render.ros
```

**Notes** — lazily rather than at construction because a large fraction of objects spawned
during a level load are never visible and would pay for a cache they never read.
