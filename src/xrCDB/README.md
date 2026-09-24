# src/xrCDB — the static collision database

Chapter 7 of [`SYSTEM-REQUIREMENTS.md`](../../SYSTEM-REQUIREMENTS.md#7-build-order).

This module answers one question, millions of times per level: *what static geometry is at
this ray / in this box / inside this frustum?* A level's world geometry is frozen once it
loads — the walls do not move — so it is compiled into one immutable spatial index built
over a triangle soup, and every subsystem that needs to know where the world is asks that
index. Bullets, footsteps, line-of-sight, sound occlusion, grass placement, light
visibility, wallmark projection and the physics engine's mesh collider all bottom out here.

The module also owns three things that grew up next to the collision database and never
left: the **frustum** type (a set of half-spaces with polygon clipping, used at least as
heavily by the renderer's visibility pass as by collision), a header of **analytic
intersection primitives** (ray/triangle, box/triangle, sphere/triangle, ray/oriented-box),
and the **dynamic** counterpart of the static tree — a loose octree of moving objects.
Putting the dynamic index in the same module as the static tree is arbitrary; they share
nothing but the frustum type. A rebuilder may split them.

This is the one seam in the whole recipe marked **buildable**
([Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)),
so this chapter teaches the rebuild rather than naming a library to shop for.

## Where it sits

It rests on the core layer only: the virtual filesystem (to read a level's collision file
and to write the build cache), the vector/matrix/plane types, the checksum, the thread
primitives, and the allocator. It is built before the material system, the engine and the
renderer, all three of which depend on it. It has one upward reference that is a layering
accident: the statistics dump reaches for the engine's debug font, and the spatial octree's
debug assertions reach for the engine's object type. Neither is load-bearing; a rebuild
should invert both.

## What is delegated and what is written here

The original **vendors a 2001-vintage collision library** (OPCODE) for exactly one job:
turning a triangle soup into a flattened binary tree of axis-aligned boxes, and reading and
writing that flattened array. Everything else — the triangle payload, the collectors that
weld a soup together, all three query traversals, the result policy, the level-file format,
the build cache, the frustum, the analytic primitives, the dynamic octree — is written
here, and the queries do not even call into the library: they walk its node array directly
with their own stabbers.

The delegated part is small enough that this chapter describes it outright, in
[`xrCDB.cpp`](xrCDB.cpp.md) under *the tree build*. A rebuilder writes it; they do not shop
for it.

## Load-bearing ideas, named once

**Triangle soup, not a mesh.** The collision world is a flat array of vertex positions plus
a flat array of triangles, each triangle three indices into the vertex array plus a packed
32-bit payload. There is no notion of an object, a surface, a smoothing group or a normal.
A triangle's identity is its index in the array, and that index is the only handle a query
result hands back — every caller that wants more looks the triangle up itself.

**The payload is a bitfield and its width is frozen.** A triangle is exactly 16 bytes:
three 32-bit vertex indices and one 32-bit word holding a **material id** (14 bits), a
**suppress-shadows** bit, a **suppress-wallmarks** bit, and a **sector id** (16 bits). The
16-byte size is not a nicety: a few hundred thousand triangles are walked at random by
every query, so the array must stay cache-dense, and the size is asserted at compile time.
The two bits are a *denormalization*: they are copies of flags that really live on the
material, cached into the triangle so the renderer's shadow and decal passes can reject a
hit without a second indirection. Everything else about a surface — whether it is
shootable, breakable, climbable, passable, how loud a footstep on it is — is reached
through the material id, in [chapter 8](../xrMaterialSystem/README.md).

**Two-sidedness is a property of the query, not of the triangle.** Nothing in the triangle
records a facing rule. Each caller decides whether its ray culls back-facing triangles, and
the choice differs by purpose: a bullet culls (it may only hit a surface it can see), a
line-of-sight test usually does not.

**Sector ids tie collision to visibility.** Each triangle names the sector it belongs to,
which is how the renderer turns a collision hit into a visibility question. Sector `0` is a
real sector, so there is no in-band "no sector" value here; the 16-bit field is simply
assumed wide enough for every level shipped.

**The tree is built once and never mutated.** Every query allocates nothing in the tree and
writes nothing to it. Only the *result buffer* and the tiny traversal state are per-query,
and those live in the caller's own collider object. This is why concurrent queries need no
lock, and it is the single most important property of the module.

**The result policy is a compile-time shape, not a runtime branch.** A query takes a set of
option bits — cull, first-only, nearest-only, full-test — and the traversal is specialized
on them before it starts, so the inner loop has no option checks in it. In a rebuild the
specialization is an optimization; the *policy* is the decision, and it is described per
query below.

**A built tree ships with the level, and is cached otherwise.** Building a tree over a few
hundred thousand triangles costs seconds. Levels therefore carry the built tree inside
their collision file when they were authored recently enough, and when they were not, the
engine builds once and writes a cache file next to the user's data, keyed by a checksum of
the source geometry so that stale caches are rejected rather than trusted.

**Two indexes, two shapes.** The static tree is a binary tree of tight boxes over
triangles. Moving objects live in a completely different structure — a loose octree of
bounding spheres, where an object's *radius* picks its depth and its *position* picks its
node, giving insertion and removal in constant time. Both are queried by ray, box and
frustum, and the game layer usually asks both and merges the results by distance.

## The files

| File | Role |
|---|---|
| [`xrCDB.h`](xrCDB.h.md) | The public surface of the static database: triangle, result, options, model, collider, collectors |
| [`xrCDB.cpp`](xrCDB.cpp.md) | The model: geometry ownership, the tree build (the delegated part, described), serialization and the build cache |
| [`xrCDB_ray.cpp`](xrCDB_ray.cpp.md) | Ray query: slab test down the tree, Möller–Trumbore at the leaves, nearest/first/all policy |
| [`xrCDB_box.cpp`](xrCDB_box.cpp.md) | Box query: box-box down the tree, separating-axis box-triangle at the leaves |
| [`xrCDB_frustum.cpp`](xrCDB_frustum.cpp.md) | Frustum query: half-space box test down the tree, polygon clipping at the leaves |
| [`xrCDB_Collector.cpp`](xrCDB_Collector.cpp.md) | Building a soup: vertex welding, duplicate-face removal, edge adjacency |
| [`Frustum.h`](Frustum.h.md) · [`Frustum.cpp`](Frustum.cpp.md) | The frustum: construction from a matrix, a portal or an occluder; sphere/box/polygon tests; polygon clipping |
| [`Intersect.hpp`](Intersect.hpp.md) | Analytic primitives: ray/triangle, box/triangle, sphere/triangle, sphere/oriented-box, ray/oriented-box |
| [`xrXRC.h`](xrXRC.h.md) · [`xrXRC.cpp`](xrXRC.cpp.md) | The per-thread query handle: a collider plus its timing counters |
| [`xr_collide_defs.h`](xr_collide_defs.h.md) | The vocabulary the game layer queries in: targets, ray definitions, merged results, the ray cache |
| [`xr_area.h`](xr_area.h.md) · [`xr_area.cpp`](xr_area.cpp.md) | The object space: owns the static model, loads the level collision file, manages the cache |
| [`xr_area_raypick.cpp`](xr_area_raypick.cpp.md) | The four ray entry points the game uses, merging static and dynamic hits |
| [`xr_area_query.cpp`](xr_area_query.cpp.md) | The oriented-box query, expressed as a six-plane frustum |
| [`ISpatial.h`](ISpatial.h.md) · [`ISpatial.cpp`](ISpatial.cpp.md) | The dynamic index: what it means to be spatially registered, and the loose octree |
| [`ISpatial_q_ray.cpp`](ISpatial_q_ray.cpp.md) | Ray query against the dynamic octree |
| [`ISpatial_q_box.cpp`](ISpatial_q_box.cpp.md) | Box and sphere queries against the dynamic octree |
| [`ISpatial_q_frustum.cpp`](ISpatial_q_frustum.cpp.md) | Frustum query against the dynamic octree |
| [`ISpatial_verify.cpp`](ISpatial_verify.cpp.md) | The octree's self-consistency check |
| [`stdafx.h`](stdafx.h.md) · [`StdAfx.cpp`](StdAfx.cpp.md) | Build-time header aggregation; no decisions |
