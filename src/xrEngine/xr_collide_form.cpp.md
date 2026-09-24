# src/xrEngine/xr_collide_form.cpp

> The per-object collision proxies a ray can hit — a skeleton's per-bone primitives rebuilt each frame from the animated pose, an authored trigger volume, and a hand-built shape list.

**Needs** — [`xr_collide_form.h`](xr_collide_form.h.md) · [`xr_object.h`](xr_object.h.md) · [`IGame_Level.h`](IGame_Level.h.md) · [`xrCDB/xr_area.h`](../xrCDB/xr_area.h.md) · [`xrCDB/Frustum.h`](../xrCDB/Frustum.h.md) · [`xrCore/FMesh.hpp`](../xrCore/FMesh.hpp.md) · [`xrCore/Animation/Bone.hpp`](../xrCore/Animation/Bone.hpp.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`xr_collide_form.h`](xr_collide_form.h.md)
**Tier floor** — T1: the per-bone primitive is an overlapped record whose three shapes share storage, and the ray tests run inside the level's query loop.

## Purpose

The [static collision database](../xrCDB/README.md) answers rays against the level's
triangles. Moving objects are not in it, and putting them in would mean rebuilding a tree
every frame. Instead every object may carry a *collision form*: a small set of primitives
the object space tests directly after it has narrowed the candidate set spatially.

Three fillings exist, and the difference between them is what a rebuild must keep:

- **A skeleton** — one primitive per visible bone, rebuilt from the animated pose. This is
  what makes a shot hit an arm rather than a box.
- **An event box** — six planes forming an oriented box, used for triggers. It answers
  "is this object inside me", not "does a ray hit me".
- **A shape list** — an authored set of spheres and boxes in the object's own space. Used
  for the restrictor volumes and other authored regions.

## State

```text
RECORD CollisionForm                    # the shared base
  owner       : Object
  kind        : {object, shape}         # the object space treats the two differently
  local_box   : axis-aligned bounds     # in the owner's space
  local_sphere: sphere                  # in the owner's space; the first-level reject

RECORD BoneElement                      # one primitive; exactly one variant is live
  kind    : {box, sphere, cylinder}
  bone    : int (16-bit)                # the invalid marker disables the element
  VARIANT box      : world-to-element transform, half extents
  VARIANT sphere   : centre and radius, in world space
  VARIANT cylinder : centre, axis, height, radius, in world space

RECORD SkeletonForm : CollisionForm
  elements      : list<BoneElement>     # ordered by bone index
  visible_mask  : int (64-bit)          # which bones were visible when the list was built
  pose_frame    : int                   # frame the elements were last posed
  bounds_frame  : int                   # frame the coarse bounds were last updated

RECORD EventBoxForm : CollisionForm
  planes : six planes, inward-facing, built once at construction

RECORD ShapeForm : CollisionForm
  shapes : list of sphere | (box transform, its inverse)
```

Invariants:

- A bone element's stored transform is **world-to-element**, not element-to-world. The ray
  test transforms the *ray* into the primitive's space and tests against an axis-aligned
  box at the origin, which is cheaper than transforming the box.
- An element whose bone index is the invalid marker is skipped by every pass. This is the
  only disable mechanism, and it is used to retire a bone whose transform could not be
  inverted.
- The visibility mask is a **64-bit word, one bit per bone**, so a skeleton with more than
  64 bones cannot have its element list invalidated correctly by bones past the 64th. That
  is a real limit and it is not checked.
- Both frame stamps exist so that a form hit by several ray queries in one frame is posed
  once. The coarse bounds and the detailed pose are stamped separately because the coarse
  test runs far more often than the detailed one.

## Two levels, and why

A ray query against a skeleton runs in two stages, and the stages have different costs and
different refresh rates:

1. **Coarse** — the object's bounding sphere, cheap, updated per frame from the visual's
   own bounds.
2. **Detailed** — every visible bone's primitive, posed from the animation, only reached
   when the coarse test passes.

The detailed stage is where the cost is, and posing it requires evaluating the skeleton —
which is itself expensive and which the renderer may not have done yet for this object this
frame. So it is deferred until a ray actually reaches it. An object nobody shoots at never
poses its collision skeleton.

## `CCF_Skeleton::BuildState` — posing the elements

**Contract** — evaluates the skeleton, and rebuilds every element's world-space primitive
from its bone's current transform. Rebuilds the *element list itself* only when the set of
visible bones has changed. Called at most once per frame per form.

```text
FUNCTION pose_elements(form)
  form.pose_frame = current_frame
  skeleton = the owner's visual as a skeleton
  skeleton.evaluate_bones()
  to_world = owner.transform

  IF skeleton.visible_bones != form.visible_mask THEN
    form.visible_mask = skeleton.visible_bones
    rebuild the element list:
      reset the coarse bounds from the visual's own
      FOR EACH bone
        SKIP if not visible, if it has no authored shape, or if the shape is marked
             not-pickable
        append an element for (bone index, shape kind)

  FOR EACH element
    SKIP if disabled
    bone_to_model = skeleton.transform_of(element.bone)
    CASE box:
      element_to_bone = the authored box's transform
      element.half_extents = the authored half extents
      world = to_world * bone_to_model * element_to_bone
      element.world_to_element = invert(world)
      IF the inversion failed THEN
        report the bone, the object and its visual; disable the element
    CASE sphere:
      element.sphere = the authored sphere, moved through bone_to_model then to_world
    CASE cylinder:
      element.cylinder = the authored cylinder, likewise, radius and height unchanged
```

**Invariants** — the not-pickable flag is authored per bone-shape and is how a model
declares a bone that exists for physics or attachment but must never stop a bullet.

A bone transform with zero scale produces a non-invertible element transform. That is a
*data* error — an animation or model bug — and the engine's response is to report it in
full and permanently disable that element rather than to fail the query. Disabling is
correct: one unhittable bone is a smaller defect than a crash or an undefined hit position.

Note the sphere and cylinder cases do **not** transform the radius. A non-uniform or scaled
bone therefore gives a sphere of the wrong size. The skeleton format assumes unscaled bones
and this is where that assumption is cashed in.

## `CCF_Skeleton::BuildTopLevel` — the coarse bounds

**Contract** — refreshes the cheap reject volume from the visual's current bounds, once per
frame.

```text
FUNCTION update_coarse(form)
  form.bounds_frame = current_frame
  v = the visual's own bounds
  form.local_box.min = midpoint(form.local_box.min, v.box.min)
  form.local_box.max = midpoint(form.local_box.max, v.box.max)
  grow form.local_box by 0.05
  form.local_sphere.centre = midpoint(form.local_sphere.centre, v.sphere.centre)
  form.local_sphere.radius = (form.local_sphere.radius + v.sphere.radius) / 2
```

**Notes** — this *averages* the previous bounds with the current ones rather than replacing
them, so the reject volume lags the visual by a one-pole filter. That is deliberate
hysteresis: an animated object's bounds jitter frame to frame, and a lagging, slightly
grown volume is stable. The growth of 0.05 world units compensates for the lag, and the two
together must be read as one decision — remove the growth and a fast-moving object's rays
start missing.

A rebuild may replace both with the visual's exact bounds and accept the jitter; it will be
correct and marginally slower.

## `CCF_Skeleton::_RayQuery`

**Contract** — tests a ray against the skeleton and appends every hit (or only the first,
or only the nearest, by the query's flags) to the result set. Each hit carries the owner,
the distance and **the bone index**, which is what lets the game resolve the hit to a body
part and a material.

```text
FUNCTION ray_query(form, query, results) -> hit
  IF bounds_frame != current_frame THEN update_coarse(form)

  sphere = form.local_sphere moved to world space
  IF the ray misses that sphere, or first touches it past the query's range THEN
    RETURN false

  IF pose_frame != current_frame THEN
    pose_elements(form)
  ELSE IF the skeleton's visible bones no longer match form.visible_mask THEN
    pose_elements(form)              # the model changed between two picks in one frame

  hit = false
  FOR EACH element
    SKIP if disabled
    test the ray against the element's primitive, honouring the query's back-face rule
    IF it hits within range THEN
      hit = true
      append (owner, distance, element.bone) to results
      IF the query wants only the first hit THEN BREAK
  RETURN hit
```

**Invariants** — the mid-frame re-pose is not defensive clutter: a bone can be hidden or
shown by game logic between two ray queries in the same frame (a limb blown off, a
detachable part), and the element list must follow. It is forced by setting the pose stamp
to the previous frame and re-posing.

The back-face rule is a query flag: with culling on, a ray starting *inside* a primitive
does not hit it; with culling off it does. Both are needed — a shot from inside a creature
should hit it, a visibility ray from inside a bush should not be stopped by it.

## Ray-versus-primitive

Three small tests, each with the same shape: intersect, and accept either an
outside-origin hit always, or an inside-origin hit only when culling is off.

- **Box** — the ray is transformed into the element's space by the stored inverse and
  tested against an axis-aligned box centred at the origin. The reported distance is
  measured in *that* space, which is only equal to the world distance because the transform
  has no scale.
- **Sphere** and **cylinder** — tested directly in world space, since both were posed
  there.

## `CCF_EventBox`

**Contract** — an oriented unit box centred on the owner, built once at construction from
the owner's transform, represented as six inward-facing planes. It answers containment, not
rays: the ray query always reports a miss.

```text
FUNCTION contains(box, other) -> bool
  centre, radius = other's visual bounding sphere, moved to world space
  FOR EACH of the six planes
    IF the signed distance from centre to the plane > radius THEN RETURN false
  RETURN true
```

**Invariants** — this is a sphere-versus-convex-volume test using only the face planes, so
it reports false positives near the box's edges and corners. For a trigger volume that is
acceptable and is the standard cheap test.

The box is built from the owner's transform **at construction time and never rebuilt**. An
event box that moves stops working. That is a real constraint on how these may be used:
they are placed by the level and stay put.

The construction enumerates the eight corners of a half-unit cube and builds the planes
from three corners each; the corner ordering is what fixes the planes' facing, and getting
it wrong inverts the volume.

## `CCF_Shape`

**Contract** — an authored list of spheres and boxes in the owner's local space. Supports
both ray queries and containment. The bounds are computed on demand by `ComputeBounds`
after the shapes are added — they are not maintained incrementally.

The ray query transforms the ray into the owner's space *once* and tests every shape there,
which is the opposite of the skeleton's approach and is right for the same reason: here the
shapes share one space, so moving the ray once beats moving each shape.

```text
FUNCTION shape_ray_query(form, query, results) -> hit
  move the ray into the owner's local space
  IF it misses the local bounding sphere THEN RETURN false
  FOR EACH shape, by index
    sphere: intersect directly
    box:    move the ray by the box's stored inverse, test a unit box at the origin,
            then convert the hit point's distance back
    IF hit within range THEN
      append (owner, distance, shape index) to results
      IF only the first is wanted THEN RETURN true
```

Each box stores **both** its transform and its inverse, computed once when it is added.
That is the trade the skeleton makes too: invert at build time, never at query time.

**The containment test has a bug worth naming**: the per-plane rejection of the box case
uses a loop-breaking construct that leaves the *shape* loop rather than skipping to the
next shape, so a shape list whose first box does not contain the point reports no
containment even when a later shape does. Spheres are unaffected. A rebuild should make
each shape's rejection continue to the next shape.

`ComputeBounds` grows the bounds over every shape, and recomputes the bounding sphere from
the box only when there is more than one shape or any box — a single sphere shape uses
itself as the bounding sphere, which is exact.

## Notes

The header declares a query-flag vocabulary (fetch triangles, boxes, spheres; first-only;
top-level-only; static; dynamic; coarse) and a result record able to carry all three shape
kinds, for a **box query that does not exist**: the declaration is commented out in all
three fillings. Only ray queries are implemented. A rebuild should implement the box query
or delete the vocabulary; carrying it forward as dead surface is how it survived this long.

The result record also carries helpers to transform a triangle or a box into world space as
it is appended, which is where the query flags' "get triangles" was meant to be served
from. They are used by the level's own queries, not by these forms.
