# src/xrCDB/xr_collide_defs.h

> The vocabulary the game layer asks collision questions in — what to test against,
> what a hit is, and how hits from the static world and from moving objects are merged.

**Needs** — [`xrCDB.h`](xrCDB.h.md) · [`xrCore/_vector3d.h`](../xrCore/_vector3d.h.md) · [`xrCore/_matrix.h`](../xrCore/_matrix.h.md)
**Used by** — [`xr_area.h`](xr_area.h.md) · [`xr_area_raypick.cpp`](xr_area_raypick.cpp.md) · [`Feel_Vision.h`](../xrEngine/Feel_Vision.h.md) · [`Rain.h`](../xrEngine/Rain.h.md) · [`xr_collide_form.h`](../xrEngine/xr_collide_form.h.md) · [`xr_efflensflare.cpp`](../xrEngine/xr_efflensflare.cpp.md) · [`xr_efflensflare.h`](../xrEngine/xr_efflensflare.h.md) · [`BlackGraviArtifact.h`](../xrGame/BlackGraviArtifact.h.md) · [`Explosive.cpp`](../xrGame/Explosive.cpp.md) · [`Explosive.h`](../xrGame/Explosive.h.md) · [`HUDTarget.cpp`](../xrGame/HUDTarget.cpp.md) · [`HUDTarget.h`](../xrGame/HUDTarget.h.md) · [`Level_Bullet_Manager.cpp`](../xrGame/Level_Bullet_Manager.cpp.md) · [`Level_Bullet_Manager.h`](../xrGame/Level_Bullet_Manager.h.md) · _and 4 more_
**Tier floor** — T2: small records and a sort; it inherits nothing lower.

## Purpose

The tree query returns triangle indices. The game asks questions like "can this creature see
that one", whose answer may be a wall, a door that is also a physics body, or another
creature — three different kinds of thing that must come back in one comparable form. This
file defines that form, and the request that produces it.

It is a header with no implementation because it is all record shapes and tiny predicates;
the code that uses them is in [`xr_area_raypick.cpp`](xr_area_raypick.cpp.md) and in the
engine's collision-form layer.

## State

```text
ENUM QueryTarget            # a set, not an alternative: a query names several
  static_world              # the collision tree
  object                    # a game object's own collision form
  shape                     # a trigger or zone volume
  obstacle                  # a volume that blocks movement but not sight

RECORD RayRequest
  origin : vec3
  dir    : vec3             # unit length; asserted
  range  : real             # must be positive and finite
  flags  : query options    # the same cull / first / nearest bits the tree query takes
  target : set<QueryTarget>

RECORD Hit
  object  : optional<GameObject>   # absent means the static world
  range   : real                   # distance along the ray
  element : int                    # triangle index for static, bone or part index for
                                   # an object; negative means "no hit"
# Invariant
#   element >= 0  <=>  this Hit is real. There is no separate validity flag.

RECORD HitSet
  hits : list<Hit>                 # the merged result of one request
```

**The two meanings of `element` are the load-bearing compromise.** A hit against the world
identifies a triangle; a hit against a creature identifies which of its parts was struck,
which is what decides damage. One field carries both because the consumer always knows which
it asked for — it has the object reference right there — and giving them separate fields
would make the sort and the "keep the nearer one" comparison, which are the operations that
actually run on mixed results, care about a distinction they do not need.

The negative-means-nothing convention is the same idea: a hit needs an ordering by range,
and a sentinel in `element` keeps the record a plain value with no optionality to carry.

## `Hit.take_if_nearer`

**Contract** — replaces this hit with another if the other is closer, and reports whether it
did. Three arms — from a tree result, from another hit, from loose fields — because the
three sources exist; they are one operation.

This is how a "nearest overall" answer is assembled from independent queries: seed a hit
with the request's full range and a negative element, then offer it every candidate. At the
end, a non-negative element means something was struck and the range is the nearest.

## `HitSet.append` · `sort` · `clear`

**Contract** — accumulate, order by range, reset. `append` has a *nearest* mode that
overwrites the last hit instead of growing the list when the new one is closer — which
collapses a whole query to one result without the caller tracking it — and an ordinary mode
that grows.

**Invariants** — after `sort`, hits are in non-decreasing range. The sort is what makes a
merged static-plus-dynamic result meaningful, since the two sources are queried
independently and each is only internally ordered.

The buffer keeps its capacity across uses on purpose: these live one per thread and are
reused thousands of times a frame.

## `RayCache`

**Contract** — remembers one previous occlusion query and its answer, plus the three vertices
of the triangle that blocked it. Answers a repeat of the same query without touching the
tree, and answers a *similar* query by testing that one remembered triangle first.

```text
RECORD RayCache
  origin, dir : vec3
  range       : real
  blocked     : bool
  verts       : vec3[3]        # the blocking triangle, if there was one

FUNCTION matches(o, d, r) -> bool
  RETURN o is within tolerance of origin
     AND dot(d, dir) is within tolerance of 1      # same direction, not merely close
     AND r is within tolerance of range
```

**This is a real algorithmic decision, not a memo.** Line-of-sight queries are issued by the
AI from the same eye position along nearly the same direction every frame, and the thing
blocking them is almost always the same wall. Testing that one remembered triangle costs one
ray-triangle test; a tree descent costs hundreds of box tests. The cache is owned by the
*asking entity*, not by the database, so each creature's line of sight to each target keeps
its own — which is what makes the hit rate high, and which is why the cache is a parameter to
the query rather than state inside it.

Comparing directions by their dot product against one, rather than componentwise, is
deliberate: it is the angular difference that matters, and two normalized directions can
differ componentwise while being the same ray for this purpose.

## `HitCallback` · `TestCallback`

**Contract** — two caller-supplied steps the general ray query invokes: one per hit as hits
are produced, returning whether to keep going; one per candidate object before testing it,
returning whether to test it at all. The first is how a caller implements "keep going until
you hit something that actually stops a bullet" without the database knowing what stops
bullets; the second is how it skips whole categories without the database knowing the
categories.
