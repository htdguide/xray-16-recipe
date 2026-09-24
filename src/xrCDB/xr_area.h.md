# src/xrCDB/xr_area.h

> Declares the object space — the level's static collision model, the moving-object
> index, and the queries that consult both.

**Needs** — [`xr_collide_defs.h`](xr_collide_defs.h.md) · [`xrXRC.h`](xrXRC.h.md) · [`xrCDB.h`](xrCDB.h.md) · [`ISpatial.h`](ISpatial.h.md) · [`Common/Noncopyable.hpp`](../Common/Noncopyable.hpp.md)
**Used by** — [`xr_area.cpp`](xr_area.cpp.md) · [`xr_area_query.cpp`](xr_area_query.cpp.md) · [`xr_area_raypick.cpp`](xr_area_raypick.cpp.md) · [`IGame_Level.cpp`](../xrEngine/IGame_Level.cpp.md) · [`IGame_Level.h`](../xrEngine/IGame_Level.h.md) · [`thunderbolt.cpp`](../xrEngine/thunderbolt.cpp.md) · [`xr_collide_form.cpp`](../xrEngine/xr_collide_form.cpp.md) · [`CalculateTriangle.h`](../xrPhysics/CalculateTriangle.h.md) · [`PHWorld.cpp`](../xrPhysics/PHWorld.cpp.md) · [`dSortTriPrimitive.h`](../xrPhysics/tri-colliderknoopc/dSortTriPrimitive.h.md) · [`dTriCylinder.cpp`](../xrPhysics/tri-colliderknoopc/dTriCylinder.cpp.md) · [`dTriSphere.cpp`](../xrPhysics/tri-colliderknoopc/dTriSphere.cpp.md)
**Tier floor** — T2: it declares per-thread storage as a first-class concept, which most
tiers have.

## Purpose

Declares the surface implemented across [`xr_area.cpp`](xr_area.cpp.md) (loading and
lifecycle), [`xr_area_raypick.cpp`](xr_area_raypick.cpp.md) (the ray entry points) and
[`xr_area_query.cpp`](xr_area_query.cpp.md) (the oriented-box query).

One decision belongs to the header itself: the query scratch — a query handle, a hit set and
a list of candidate objects — is declared as **per-thread** storage, not as fields of the
object space. That is the concurrency contract of this module made concrete. The object
space is a shared, read-only-after-load thing; the mutable state a query needs is the
thread's, so two threads querying the same level share nothing and lock nothing.

## State

Declares the shape; the record, the per-thread scratch and the level collision file's
layout are in [`xr_area.cpp`](xr_area.cpp.md).

## Exported units

- **`ObjectSpace`** — owns the level's static collision model, holds a reference to the
  moving-object index, and exposes every query the game layer uses.
- **`load`** — three arms: from the level's default collision file, from a named file, from
  an already-open reader.
- **`create`** — builds or restores the model from geometry plus a header, taking the four
  hooks (build fix-up, serialize extra, deserialize check, material remap).
- **`ray_test`** — *is anything in the way*. Takes a ray cache.
- **`ray_pick`** — *what is the nearest thing in the way*, static and dynamic merged.
- **`ray_query`** — the general one: all hits, ordered, with a per-hit callback that decides
  whether to continue. Also a form that tests one named collision form instead of the world.
- **`box_query`** — an oriented box against the static world, returning the triangles.
- **`nearest`** — the game objects within a radius of a point; three arms differing only in
  whose scratch buffer they use.
- **accessors** — the static triangles, the static vertices, the model itself, and the
  level's bounding volume.

## Notes

The object space is declared non-copyable and holds the model *by value*, which makes the
level's whole collision world a member rather than a pointer. That is deliberate: its
lifetime is exactly the level's, and there is no case where one outlives the other.

The debug rendering member and its factory pointer exist only in instrumented builds and
reach into the renderer's interface layer. That is instrumentation, not structure.
