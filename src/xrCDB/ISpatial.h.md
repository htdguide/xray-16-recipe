# src/xrCDB/ISpatial.h

> What it means for a thing to have a place in the world — the record every
> spatially indexed object carries, the interface it implements, and the octree that holds
> them all.

**Needs** — [`xrCDB.h`](xrCDB.h.md) · [`xrCore/_sphere.h`](../xrCore/_sphere.h.md) · [`xrCore/xrPool.h`](../xrCore/xrPool.h.md) · [`Common/Noncopyable.hpp`](../Common/Noncopyable.hpp.md)
**Used by** — [`light.cpp`](../Layers/xrRender/light.cpp.md) · [`light.h`](../Layers/xrRender/light.h.md) · [`light_gi.h`](../Layers/xrRender/light_gi.h.md) · [`r__dsgraph_build.cpp`](../Layers/xrRender/r__dsgraph_build.cpp.md) · [`r__dsgraph_structure.h`](../Layers/xrRender/r__dsgraph_structure.h.md) · [`ISpatial.cpp`](ISpatial.cpp.md) · [`ISpatial_q_box.cpp`](ISpatial_q_box.cpp.md) · [`ISpatial_q_frustum.cpp`](ISpatial_q_frustum.cpp.md) · [`ISpatial_q_ray.cpp`](ISpatial_q_ray.cpp.md) · [`ISpatial_verify.cpp`](ISpatial_verify.cpp.md) · [`xr_area.cpp`](xr_area.cpp.md) · [`xr_area.h`](xr_area.h.md) · [`xr_area_raypick.cpp`](xr_area_raypick.cpp.md) · [`ICollidable.cpp`](../xrEngine/ICollidable.cpp.md) · _and 9 more_
**Tier floor** — T2: it is an interface plus a tree of small nodes; the pooling of nodes is
a performance choice, not a requirement.

## Purpose

This is an interface header, so it is substantive: what it demands of an implementor *is*
the contract a rebuild must satisfy. The static tree indexes triangles that never move. This
indexes everything that does — creatures, items, lights, particle systems, sound emitters,
trigger volumes — and the whole design follows from one requirement the file states outright
in a comment: **insertion and removal must be constant time**, because objects move every
frame and a rebuild-the-tree index would dominate the frame.

The consequence is a *loose octree* with an unusual placement rule: an object's **radius
alone determines its depth** and its **position alone determines its node** at that depth.
No comparison, no balancing, no rebuild — two divisions and three sign tests find the slot.

## State

```text
RECORD SpatialData                 # every indexed object carries one
  type        : set<Kind>          # what this object is, for query filtering
  sphere      : (centre, radius)   # the bound the index actually uses
  node_centre : vec3               # cached: the centre of the node it currently sits in
  node_radius : real               # cached: that node's half-size, doubled
  node        : optional<Node>     # cached: which node holds it; absent means unregistered
  sector      : int                # which visibility sector it is in
  space       : reference to the index it belongs to

# Invariants
#   node present  <=>  the object is in exactly one node's item list
#   the object's sphere fits inside the cached node's bounds -- this is the whole
#     placement invariant, and `is_placed_correctly` below is its test
#   sector is stale whenever the "sector invalid" kind bit is set
```

```text
ENUM Kind                # a set, not an alternative
  renderable             # the render graph wants it
  light                  # and its hemispherical variant
  collideable            # ray and box queries against objects find it
  visible_to_ai          # the AI's vision system considers it
  reacts_to_sound        # the AI's hearing system delivers to it
  physical               # has a rigid body
  obstacle               # blocks movement
  shape                  # a trigger or zone volume
  sector_invalid         # not a kind: a dirty flag meaning `sector` needs recomputing
```

**The kind set is the query filter and it is why one index serves every subsystem.** The
renderer asks for renderables and lights, the AI asks for things visible to it, collision
asks for collideables. Storing every object once and filtering by a bit mask at query time
is cheaper than five indexes each maintaining its own membership — and crucially it is
cheaper *to move an object*, which is the operation that happens every frame.

Packing a dirty flag into the same set as the kinds is the one wart. It works because the
flag's bit is far above the kind bits and no query masks against it, but it means "what kind
of thing is this" and "is its sector stale" are one field. A rebuild should separate them.

```text
RECORD Node
  parent   : optional<Node>
  children : optional<Node>[8]     # octants, indexed by the sign bits of (x, y, z)
  items    : list<Spatial>         # objects that live at this node's level

RECORD Index
  root          : Node
  centre        : vec3             # the world's centre
  bounds        : real             # the world's half-size
  lock          : mutex
  node_pool     : pool of Node
  free_nodes    : list<Node>
  stats         : counts and timers
```

**The octant index is the three sign bits of the offset from the node's centre**, in the
order x, y, z from least significant — so a child's index and the direction from parent to
child are the same three bits, and the table mapping index to direction is trivially
consistent with the index computation. The file asserts that consistency at insertion time,
which is the right instinct: getting it wrong puts objects in the wrong octant and they
become findable only by queries that happen to visit both.

**`bounds` is a half-size and `node_radius` is that doubled**, and the factor of two is the
"loose" in loose octree: a node's *test* volume is twice its *placement* volume, so an
object may sit in a node while overhanging it by up to its own radius. That is what allows
an object to be placed by position alone without checking whether it fits — it always fits,
because the test volume was sized for the worst case.

## `Spatial` — the interface

**Contract** — what an indexable thing must provide.

```text
INTERFACE Spatial
  data()                  -> SpatialData     # the record above, by reference
  is_placed_correctly()   -> bool            # does my sphere still fit my cached node
  register()                                 # enter the index
  unregister()                               # leave it
  moved()                                    # my sphere changed; fix my placement
  sector_point()          -> vec3            # the point whose sector is mine
  set_sector(id)                             # accept a recomputed sector
  as_game_object()        -> optional<...>   # the four downcasts
  as_sound_feeler()       -> optional<...>
  as_renderable()         -> optional<...>
  as_light()              -> optional<...>
```

**The four downcasts are the load-bearing wart.** A query returns spatial handles, and every
consumer immediately needs to know whether this is a game object, a light, a renderable or a
sound listener — so the interface answers all four directly instead of letting callers
attempt a dynamic type test, which in the original's tier is both slow and unavailable in
some builds. The decision underneath is *a query result must be resolvable to its concrete
role without a type test*; a rebuild with cheap sum types or trait objects expresses that
without four methods, but must still express it, because the alternative — one index per
role — was rejected above.

`sector_point` exists because an object's sector is decided by a single representative
point, and for most objects that is the centre of its bound but for some it is not.

## `SpatialBase` — the default implementation

**Contract** — supplies the registration lifecycle so that an implementor only writes the
downcasts. Substance in [`ISpatial.cpp`](ISpatial.cpp.md).

## `Index` — the octree

**Contract** — insert, remove, query. Substance in [`ISpatial.cpp`](ISpatial.cpp.md) and the
three query files. Options on a query are *first-only*, *nearest-only* and *ordered* (the
last is declared and, in the shipped traversals, not honoured — results come back in
traversal order and a caller wanting distance order sorts).

- `ray(out, options, mask, origin, dir, range)` — [`ISpatial_q_ray.cpp`](ISpatial_q_ray.cpp.md)
- `box(out, options, mask, centre, half_sizes)` — [`ISpatial_q_box.cpp`](ISpatial_q_box.cpp.md)
- `sphere(out, options, mask, centre, radius)` — the box query over the sphere's bound
- `frustum(out, options, mask, frustum)` — [`ISpatial_q_frustum.cpp`](ISpatial_q_frustum.cpp.md)
- `verify()` — [`ISpatial_verify.cpp`](ISpatial_verify.cpp.md)

**The mask means different things in different queries and this is a genuine trap.** The box
and frustum queries admit an object whose kind set shares *any* bit with the mask; the ray
query admits one whose kind set contains *every* bit of the mask. The difference is not
documented anywhere but the parameter names, and callers depend on both readings — the ray
callers pass compound masks meaning "collideable *and* an obstacle" while the box callers
pass a single bit. A rebuild should give the two forms different names.

## Notes

The minimum node size is a fixed world-space constant: subdivision stops when a node's
half-size reaches it, and everything smaller than that lands in a leaf at that scale. It
is the resolution floor of the whole index, chosen to be a few times a human's size, and no
derivation for the exact value is recoverable — treat it as *the scale below which
subdividing stops paying*.
