# src/Include/xrRender/ObjectSpaceRender.h

> Debug visualization of the collision query layer: a buffer of spheres, drawn once a frame.

**Needs** — [`RenderFactory.h`](RenderFactory.h.md) · [`xrEngine/ObjectSpace.h`](../../xrEngine/ObjectSpace.h.md)
**Used by** — [`RenderFactory.h`](RenderFactory.h.md) · [`dxObjectSpaceRender.cpp`](../../Layers/xrRender/dxObjectSpaceRender.cpp.md) · [`dxObjectSpaceRender.h`](../../Layers/xrRender/dxObjectSpaceRender.h.md)
**Tier floor** — T2: an accumulator and a draw; debug builds only.

## Purpose

The collision query layer answers "what is here" against the static collision database and the dynamic objects. When it is being debugged, every query it performs wants to leave a visible mark. This is that channel: queries push spheres, the frame draws them.

One instance per process, created and destroyed through the [render factory](RenderFactory.h.md), and only in a debug build.

## State

```text
RECORD ObjectSpaceRenderState
  pending : list<(Sphere, colour : int (32-bit))>   # cleared by every render
  material: Material
```

## `IObjectSpaceRender`

```text
FUNCTION reserve(count : int)             # size the buffer before a burst of adds
FUNCTION add_sphere(sphere, colour : int (32-bit))
FUNCTION set_material()
FUNCTION render()                         # draw and clear
FUNCTION copy(other)
```

**Contract** — `add_sphere` accumulates; `render` draws everything accumulated and empties the buffer. `set_material` selects the debug material to draw with — it takes no argument because there is exactly one.

**Notes** — `reserve` exists because a single collision query can push hundreds of spheres and growing the buffer inside the query's inner loop was measurable even in a debug build. It is a performance affordance in debug-only code, which is unusual enough to be worth noting: the visualization has to be cheap enough not to change the behaviour it is visualizing.

A sphere is the only shape. Every query result — a ray hit, a box overlap, an object's bounds — is reported as a sphere at a position with a colour that encodes what kind of result it was. The colour code is at the call sites, not here.
