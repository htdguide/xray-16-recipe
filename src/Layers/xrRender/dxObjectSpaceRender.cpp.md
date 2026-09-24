# src/Layers/xrRender/dxObjectSpaceRender.cpp

> The debug view of the collision world: draws the boxes the last collision query touched, and any spheres callers have queued.

**Needs** — [`dxObjectSpaceRender.h`](dxObjectSpaceRender.h.md) · [`Include/xrRender/ObjectSpaceRender.h`](../../Include/xrRender/ObjectSpaceRender.h.md) · [`xrEngine/xr_collide_form.h`](../../xrEngine/xr_collide_form.h.md) · [`R_Backend.h`](R_Backend.h.md) · [`ResourceManager.h`](ResourceManager.h.md)
**Used by** — [`dxObjectSpaceRender.h`](dxObjectSpaceRender.h.md)
**Tier floor** — T2: it issues shape draws through the debug drawing surface. Debug-only.

## Purpose

The collision system records, in a debug build, every oriented box it tested during the last query. This file draws them, so that "why did the bullet miss" has a picture. It also accepts spheres queued by any caller — bone-level hit zones, audible radii — and draws those too.

Both lists are **consumed by drawing**: a shape is queued, drawn once, and gone. That is what makes the view show the current frame's queries rather than an accumulating mess, and it is why both lists are cleared at the end of the draw and not at the start of the frame.

## State

```text
RECORD ObjectSpaceRenderer
  wireframe_material : Material                  # the named debug wireframe material
  collision_query    : CollisionQueryRecord      # the box list the collision system fills
  queued_spheres     : list<(Sphere, colour)>
```

**Invariants** — neither list is guarded. The collision system fills the box list from whichever thread ran the query, and the draw empties it from the render thread. The source marks both fields as dangerous under concurrency and does nothing about it; a rebuild that runs collision off the render thread must either lock or double-buffer. This is a real hazard, not a formality — a query completing mid-draw corrupts the list.

## `render()`

**Contract** — draws both lists and empties them. Asserts that debug drawing is enabled. Binds the wireframe material once.

```text
FUNCTION render()
  set_material(wireframe_material)

  FOR EACH box IN collision_query.boxes
    transform = box.orientation_and_position
    draw_wire_box(transform, box.half_extents, red)
    draw_wire_ellipse(transform * scale(box.half_extents), blue)
  collision_query.boxes.clear()

  FOR EACH (sphere, colour) IN queued_spheres
    transform = scale(sphere.radius) then translate to sphere.centre
    draw_wire_ellipse(transform, colour)
  queued_spheres.clear()
```

**Invariants**

- Each collision box is drawn **twice**, as a red box and as the blue ellipsoid inscribed in it. That is not decoration: the collision system tests some shapes as boxes and some as the inscribed ellipsoid, and seeing both is the only way to tell which volume actually rejected a ray.
- The ellipse transform is the box's own orientation composed with a scale by its half extents, in that order. Reversing the composition orients the ellipsoid in world axes instead of the box's, which looks almost right and is wrong.
- The sphere transform scales by the radius *and then* translates. A sphere is symmetric so the order is harmless here, but the scale is by the radius, not the diameter — the unit sphere primitive is radius 1 (see [`du_sphere.cpp`](du_sphere.cpp.md)).

## `queue_sphere(sphere, colour)` / `reserve_spheres(count)`

**Contract** — append one sphere to the pending list; reserve capacity for a known batch. Both are trivial, and the reserve exists so that a caller about to queue a creature's whole skeleton does not grow the list once per bone.

## `set_material()`

**Contract** — binds the wireframe material without drawing anything, for a caller that wants to issue its own shapes in the same style.

## Construction and destruction

**Contract** — the wireframe material is created from a fixed name in the shipped data — the debug wireframe shader with no texture — and released on destruction. The whole class exists only in debug builds.
