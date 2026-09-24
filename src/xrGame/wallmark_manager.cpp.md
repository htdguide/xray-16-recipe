# src/xrGame/wallmark_manager.cpp

> Sprays a burst of decals onto every static surface near a point — the blood and scorch an explosion leaves on its surroundings.

**Needs** — [`wallmark_manager.h`](wallmark_manager.h.md) · [`Level.h`](Level.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [`xrPhysics/CalculateTriangle.h`](../xrPhysics/CalculateTriangle.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`wallmark_manager.h`](wallmark_manager.h.md)
**Tier floor** — T1: sweeps the collision database's raw triangle and vertex arrays by index and hands triangles straight to the renderer

## Purpose

A single point in the world needs to mark *everything around it*, not one surface along one
ray. An explosion or a burst of gore does not hit one wall; it hits the floor, the two
walls in the corner and the underside of the ledge, all at once. This file answers that with
a box query against the static collision database followed by a per-triangle closest-point
test, rather than with a fan of rays — a fan would need hundreds of rays to find the same
surfaces and would still miss the ones edge-on to the centre.

The output is `wallmark` records handed to the renderer, which owns their lifetime and their
fade. Nothing here is persistent: decals are cosmetic and are not saved.

## State

```text
RECORD WallmarkManager
  wallmarks : ref wallmark-array      # the pool of decal appearances to draw from,
                                      # created through the renderer's factory
  pos       : position                # the burst centre, set by the public entry point
  owner     : optional<ref game object>  # excluded from the occlusion ray
```

**Invariants** — the decal-appearance array is a renderer-side object: the game layer names
appearances by string and never sees what one is. That indirection is load-bearing rather
than incidental, because which appearance a decal uses is chosen by the renderer from the
array at draw time, not here.

## `PlaceWallmarks`

**Contract** — the public entry point. Records the burst centre, loads the decal appearance
list from the fixed configuration section `explosion_marks`, and runs the placement sweep
synchronously. Allocates renderer-side decal records. Touches the collision database, so it
must run where that is safe — it is called from the simulation, not from a worker.

**Notes** — The configuration section is hard-coded rather than read from the owner's own
section, so **every caller gets explosion decals regardless of what caused the burst**. The
per-owner lookup exists in the source as dead code, which says the per-owner variant was
intended and abandoned. A rebuild should parameterize the section; nothing depends on the
constant.

The placement was also once queued onto the engine's parallel-work list and is now run
inline. The sweep is bounded by a configured decal count, so its cost is bounded, but it is
a box query plus a ray test per candidate triangle in the frame that fires it. A rebuild
with a job system should put it back on one.

## `StartWorkflow`

**Contract** — the sweep. Reads three tuning numbers from `explosion_marks`: the radius to
search, the decal size, and the maximum number of decals to place. Queries the static
collision database for every triangle in the axis-aligned box of that radius around the
centre, then places a decal on each accepted triangle until the count is reached. No return
value; the effect is entirely on the renderer's decal set.

**Invariants** — at most `max_count` decals are placed. The sweep visits candidate triangles
in whatever order the collision database returns them, so **which** surfaces get marked when
the cap binds is not defined by distance or by anything else meaningful. That is visible in
gameplay when a burst happens in a triangle-dense corner and marks the wrong faces; it is a
defect, not a design, and a rebuild that sorts candidates by distance before capping is
strictly better.

```text
FUNCTION place_wallmarks_around(centre)
  radius, size, max_count := config("explosion_marks")
  candidates := static_collision.triangles_in_box(centre, radius on each axis)

  placed := 0
  FOR EACH tri IN candidates
    IF placed >= max_count THEN BREAK

    # closest point on this triangle to the centre, in barycentric coordinates
    (distance, s, t, closest, direction) := closest_point_on_triangle(centre, tri)

    # is the triangle actually reachable, or is something between us and it
    IF distance > epsilon
      IF static_collision.ray_hits(centre, direction, distance - epsilon, ignoring owner)
        CONTINUE

    # reject a closest point that lies on an edge or a corner of the triangle
    IF s == 0 OR t == 0 OR s == 1 OR t == 1 THEN CONTINUE

    IF distance <= radius
      renderer.add_static_wallmark(wallmarks, closest, size, tri)
      placed := placed + 1
```

**Notes** — **The edge rejection is the subtle rule.** A closest point that lands exactly on
a triangle's edge or vertex means the centre is *outside* the triangle's own slab: the
nearest thing on that face is its border, and the face the burst is really in front of is a
neighbour. Placing there produces a decal hanging off the edge of a surface, half in the
air. Rejecting all four boundary cases costs nothing — the neighbouring triangle that the
point genuinely faces is in the same candidate set and will be accepted instead.

The occlusion ray is shortened by an epsilon before being cast, because a ray of exactly the
distance to the surface hits the surface itself. The owner is excluded so that an entity
does not shield the ground under its own feet.

The box query uses the box *inscribing* the radius on each axis rather than a sphere, so
triangles out to the box's corners are considered and then rejected by the distance test at
the end. That is the cheap ordering: the box query is the database's native operation and
the distance test is already computed.

## `AddWallmark`

**Contract** — place one decal at a point along a ray, given the triangle index the ray
struck. Used by the ray-based path rather than the sweep. Consults the material of the
struck triangle and **places nothing unless that material is flagged as accepting blood
marks**. Places nothing if the appearance array is empty.

```text
FUNCTION add_wallmark(direction, start, range, size, appearances, triangle_index)
  tri      := static_collision.triangle(triangle_index)
  material := material_library.by_index(tri.material)
  IF NOT material.accepts_bloodmarks THEN RETURN
  IF appearances is empty THEN RETURN
  end_point := start + direction * range
  renderer.add_static_wallmark(appearances, end_point, size, tri)
```

**Notes** — The material gate is the only place in this file that consults the material
table, and it is why blood does not appear on water, grass or glass. The sweep path does
*not* apply the gate, which is an inconsistency: an explosion marks any surface, a blood
spray marks only surfaces authored to take blood.

The impact point is reconstructed from the origin, the direction and the reported range
rather than taken from the collision result. The two agree, and the reconstruction is what
lets the caller pass a modified range.

## `Load`

**Contract** — reads the named configuration section's `wallmarks` key, splits it as a
delimited list of appearance names, and appends each to the decal-appearance array. The list
must be non-empty. Appends rather than replaces, so calling it twice doubles the pool.

## `Clear`

**Contract** — empties the appearance array. Does not retract decals already handed to the
renderer; those are the renderer's and fade on its own schedule.

## Closest point on a triangle

**Contract** — the geometric primitive the sweep rests on. Given a point and a triangle,
returns the squared-then-rooted distance to the nearest point of the triangle, that point
itself, the unit direction from the query point toward it, and the point's two barycentric
coordinates. Pure.

**Invariants** — the two barycentric coordinates are each in `[0, 1]` and sum to at most
one; the caller's edge rejection depends on them being *exact* at the boundary, which means
the clamping inside must assign the literal boundary value rather than something within
epsilon of it.

**Notes** — This is the standard closed-form solution: minimize the squared distance as a
quadratic in the two barycentric coordinates, and dispatch on which of the seven regions
outside the triangle — three vertex regions, three edge regions — the unconstrained minimum
falls into, clamping to that region's boundary. The case analysis is long and entirely
mechanical; a rebuild should take it from any computational-geometry reference rather than
transcribing it, and the only thing this file adds to the textbook version is returning the
barycentric coordinates so the caller can detect a boundary hit.

It is a free function in this file rather than a shared utility, which is arbitrary — the
same routine appears elsewhere in the physics layer. A rebuild should have one.
