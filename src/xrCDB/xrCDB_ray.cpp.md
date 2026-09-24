# src/xrCDB/xrCDB_ray.cpp

> Shoots a ray at the static collision tree and collects the triangles it pierces,
> under one of three result policies.

**Needs** — [`xrCDB.h`](xrCDB.h.md) · [`xrCore/_fbox.h`](../xrCore/_fbox.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: the descent is the hottest loop in the engine and is written against a
4-wide float unit with explicit control over how infinities and not-a-numbers propagate.

## Purpose

The ray query. This is the single most-called entry point in the module: bullets,
line-of-sight, footstep probes, camera collision, grass and decal placement, light
visibility and the physics engine's mesh collider all come through here.

It is its own file because the traversal is specialized on the result policy — the policy
decisions are taken before the descent starts, not inside it — and because it carries two
implementations of the box test that must produce compatible answers.

## State

Stateless between calls. The traversal's working state is a value the query creates and
drops, and the results go into the caller's collider:

```text
RECORD RayTraversal
  results   : reference to the caller's result buffer
  verts     : reference to the model's vertices     # borrowed; never written
  tris      : reference to the model's triangles    # borrowed; never written
  origin    : vec3
  direction : vec3          # assumed unit length by every caller
  inv_dir   : vec3          # 1/direction, componentwise; see the infinity note
  range     : real          # shrinks during a nearest-only query
  range_sq  : real          # kept in step with range
```

`inv_dir` is precomputed because the slab test divides by the direction at every one of the
hundreds of nodes a descent touches. A zero component makes it infinite, which is correct
and is *relied upon*: an axis-parallel ray must be rejected by a slab it misses and
accepted by one it lies inside, and the infinities give exactly that — provided the
comparison order never multiplies an infinity by a zero difference. Where the arithmetic
cannot be trusted to do that, the infinity is replaced by zero before the descent, and the
box test compensates. This is the one genuinely delicate numerical decision in the file and
a rebuild must reproduce the intent: *a ray parallel to a slab is inside it if and only if
its origin is between the planes.*

## `Collider.ray_query`

**Contract** — intersects a ray against a model and fills the collider's result buffer.
Takes the model, an origin, a *unit* direction and a maximum range, plus option bits. Waits
for the model to be ready, clears the result buffer, and descends. Allocates only if the
result buffer must grow. Reads nothing mutable, writes nothing shared: safe to run
concurrently against one model from any number of threads, each with its own collider.

**Invariants** — every reported hit has `0 < distance <= range` at the moment it is
recorded; under nearest-only the buffer holds at most one hit and it is the closest.

```text
FUNCTION ray_query(model, origin, direction, range, options) -> ()
  AWAIT model ready
  clear results
  # policy is fixed here, once, and the descent below contains no option tests
  descend(model.tree.root, policy = options)
```

### the descent

```text
FUNCTION descend(node)
  IF NOT ray_hits_box(node.center, node.extents) : RETURN
  IF distance to that box > range: RETURN        # the range prune: this is what makes
                                                 # a nearest-only query fast

  IF node.pos is a leaf: test_triangle(node.pos.triangle)
  ELSE                 : descend(node.pos)

  IF policy is first-only AND results is non-empty: RETURN   # early out

  IF node.neg is a leaf: test_triangle(node.neg.triangle)
  ELSE                 : descend(node.neg)
```

**The descent does not order its children by distance.** It always visits the
above-the-plane child first. For a nearest-only query, ordering the two children by which
slab the ray enters first would prune far harder, and this is the most obvious available
improvement in the module: the range shrinks only when a *nearer* triangle happens to be
found, and visiting in build order finds them in arbitrary order. A rebuild should sort the
two children by entry distance; the shape of the node makes that cheap, since the slab test
already computes the entry parameter.

### the box test

Two implementations, chosen once per query by what the machine can do.

The wide version computes, for all three axes at once, the parameters at which the ray
crosses each slab's two planes, takes the largest entry and smallest exit, and reports a
hit when the exit is non-negative and not before the entry — reporting the entry parameter
as the distance. The min/max order is chosen so that the not-a-number produced by
`infinity * 0` (an axis-parallel ray whose origin lies exactly on a slab plane) is filtered
out rather than poisoning the comparison; this is the reason the code cannot be written as
the obvious three-line slab test.

The scalar version instead finds, per axis, the *candidate plane* the ray could enter
through, takes the farthest of the three, and verifies that the point at that parameter is
inside the box on the other two axes — reporting that point. If the origin is inside the
box it reports the origin. It returns a point where the wide version returns a parameter,
which is why the caller's range prune is written twice: squared distance from origin to the
returned point in one case, the parameter directly in the other. A rebuild should pick one
form — the parameter — and have both paths produce it.

### the triangle test

Möller–Trumbore: build two edge vectors from the first vertex, cross the ray direction with
the second edge, and take the determinant against the first. Under **cull**, a
non-positive determinant means the triangle faces away and is rejected outright, and the
barycentric bounds are tested against the un-normalized determinant so the reciprocal is
computed only for a surviving hit. Without cull, a determinant near zero either way means
the ray lies in the triangle's plane and is rejected, and the reciprocal is taken up front.

A hit is kept only when its distance is strictly positive and within the current range —
strictly positive because a ray starting exactly on a surface must not hit that surface,
which is what makes a chained "next hit along this ray" query terminate.

### the result policy

Three policies, and they are not interchangeable:

- **all hits** — every pierced triangle, in traversal order, no ordering guarantee. The
  caller sorts if it cares. This is what the general ray query in
  [`xr_area_raypick.cpp`](xr_area_raypick.cpp.md) asks for.
- **nearest only** — the buffer holds at most one hit. When a nearer triangle is found it
  *overwrites* the existing one and the traversal's range shrinks to that distance, so
  every subsequent box test and triangle test prunes against the best answer so far. This
  self-tightening is the whole performance argument for the policy: a nearest query over a
  large level touches a small fraction of the nodes an all-hits query does.
- **first only** — stops at the first hit found anywhere, which is *not* the nearest. Used
  only for occlusion questions — "is anything at all in the way" — where the identity of
  the blocker does not matter. Cheapest of the three.

**cull** is orthogonal to all three and is a property of the caller's intent, not of the
geometry: nothing in a triangle says whether it is one- or two-sided. A projectile culls;
a line-of-sight probe from inside a building usually does not.

## Notes

The original expands the query into sixteen separately compiled traversals — every
combination of wide-or-scalar, cull, first-only, nearest-only — and dispatches through a
nested chain of option tests before descending. That is a mechanism, and the decision under
it is simply *no option may be tested inside the descent*, because the descent runs
hundreds of times per query and the options are constant for the whole query. Any technique
that achieves that satisfies the rebuild; a rebuild that cannot specialize should at least
hoist the tests out of the inner recursion.

The descent issues a prefetch for the node it has not looked at yet while testing the one
it has. That is a hardware hint with no semantics; it survives only as the observation that
the two children are the next thing touched and a rebuild may say so if its target lets it.
