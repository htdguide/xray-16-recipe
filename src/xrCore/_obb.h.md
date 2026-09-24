# src/xrCore/_obb.h

> The oriented bounding box: a rotation, a centre and three half-extents — the tight bound used where an axis-aligned box would be mostly empty, and the ray test that goes with it.

**Needs** — [`_matrix33.h`](_matrix33.h.md) · [`_matrix.h`](_matrix.h.md) · [`_vector3d.h`](_vector3d.h.md) · [`_std_extensions.h`](_std_extensions.h.md)

**Used by** — [`Intersect.hpp`](../xrCDB/Intersect.hpp.md) · [`Bone.cpp`](Animation/Bone.cpp.md) · [`Bone.hpp`](Animation/Bone.hpp.md) · [`BoneEditor.cpp`](Animation/BoneEditor.cpp.md) · [`vector.h`](vector.h.md) · [`xr_collide_form.h`](../xrEngine/xr_collide_form.h.md)

**Tier floor** — T1: fifteen contiguous reals passed by value through picking and collision code.

## Purpose

An axis-aligned box around a rotated object is loose by up to a factor of three in volume. An
oriented box is tight by construction and costs one change of frame per query. The engine
uses it for bone collision proxies, for weapon and vehicle bounds, and for the camera's
collision volume — everywhere a shape is elongated and does not happen to line up with the
world axes.

## State

```text
RECORD OrientedBox
  rotation  : Matrix3              # the three basis rows of the box's own frame
  centre    : (real, real, real)
  half_size : (real, real, real)   # HALF the extent along each of its own axes
```

**Invariants**

- The rotation's rows must be orthonormal. Nothing checks, and the ray test transposes
  rather than inverts, so a non-orthonormal rotation produces silently wrong answers rather
  than a failure.
- `half_size` is half the extent, not the extent. The box spans `centre ± rotation_row[a] *
  half_size[a]` along each of its own axes.
- The **invalidated** box is the identity rotation at the origin with *zero* half-extents —
  a degenerate point, not an accumulation identity. Unlike the axis-aligned box, this type
  has no "extend me by a point" operation and therefore needs no inverse-infinite identity.
- The **identity** box is the unit cube: identity rotation, origin, half-extents of one half.
  Same convention as the axis-aligned box, and for the same reason — a box's full transform
  is defined as the map that takes the unit cube to it.

## `xform_get` and `xform_set` — the rigid transform view

**Contract** — Read the box's rotation and centre out as a 4×4 affine transform, or write
them back from one. The half-extents are not touched by either. Allocation-free.

**Notes** — Reading out writes the fourth column as `(0, 0, 0, 1)` explicitly, producing a
proper affine transform; writing in ignores it. The pair exists so an oriented box can be
moved by the same transform arithmetic as everything else without the box type needing to
know how to compose.

The rotation is copied row-for-row in both directions, not transposed — so this pair is
consistent with itself and inconsistent with the [3×3 type's convention
warning](_matrix33.h.md#the-convention-warning). It works because the ray test below reads
the rotation as three basis *rows* via dot products, which is the 4×4 convention, not the
3×3 type's.

## `xform_full` — the unit-cube map

**Contract** — Produces the affine transform that maps the unit cube onto this box: the
rigid transform composed with a scale by the half-extents. This is what a debug renderer
draws a unit cube through, and what a shader is handed to place a box-shaped volume.

## `transform` — move a box by a transform

**Contract** — Writes this box as another box moved by an affine transform. The half-extents
are copied unchanged, so this is correct only for a **rigid** transform — one with no scale
and no shear. Feeding a scaling transform produces a box whose rotation rows are no longer
unit length and whose extents are still the original's, which is wrong in two ways at once.
Nothing checks.

**Notes** — Marked "unoptimized" in the original because it round-trips through a 4×4
composition rather than multiplying three rows directly. That is a speed note, not a
correctness one, and it tells a rebuilder that this is not on a hot path.

## `intersect` — ray against the oriented box

**Contract** — Takes a ray origin and direction in world space and a caller-held current best
distance. Reports whether the box was hit *closer than* that distance, narrowing it in place
when so. Pure, allocation-free, thread-safe against a shared immutable box.

The algorithm is the slab test in the box's own frame, which is the only reason an oriented
box is cheap: once the ray is expressed in that frame, the box is axis-aligned and the test
is the ordinary one.

```text
FUNCTION ray_hits_oriented_box(origin, direction, best_distance) -> bool
  # Change frame: the rotation is orthonormal, so its inverse is dotting
  # against its rows. No matrix inverse is computed anywhere.
  offset = origin - centre
  local_origin    = ( dot(offset, rotation.row1),
                      dot(offset, rotation.row2),
                      dot(offset, rotation.row3) )
  local_direction = ( dot(direction, rotation.row1),
                      dot(direction, rotation.row2),
                      dot(direction, rotation.row3) )

  # The slab interval starts as the whole forward ray and is clipped by
  # each of the six faces in turn.
  enter = 0
  leave = largest representable real
  IF NOT clip_against_all_six_faces(local_origin, local_direction,
                                    half_size, enter, leave) THEN
    RETURN false

  IF enter > 0 THEN                       # the ray starts outside
    IF enter < best_distance THEN best_distance = enter ; hit = true
    IF leave < best_distance THEN best_distance = leave ; hit = true
  ELSE                                    # the ray starts inside
    IF leave < best_distance THEN best_distance = leave ; hit = true
  RETURN hit
```

**Invariants**

- The interval starts at zero, not at minus infinity, so only the forward half of the ray is
  considered. An origin inside the box therefore yields an entry parameter of zero and an
  exit parameter that is the distance to the far face — which is how "inside" is detected,
  without a containment test.
- The clip reports a hit only when the interval was actually *narrowed*. An interval left
  untouched by all six faces means the direction was parallel to every axis, which cannot
  happen for a real direction but does happen for a zero one, and this is the guard against
  it reporting a spurious hit.

**Notes** — The branch after the clip has a quirk a rebuild should not copy. When the origin
is outside, the routine narrows to the *entry* parameter and then checks the exit parameter
against the newly narrowed distance as well — which can only succeed when the exit is nearer
than the entry, that is, never for a well-formed interval. The second test is dead. A rebuild
narrows to the entry when outside and to the exit when inside, and stops there.

### The slab clip

**Contract** — Clip a parameter interval against one half-space, reporting whether anything
survives. Six calls, one per face, chained so that the first failure short-circuits.

```text
FUNCTION clip(denominator, numerator, enter, leave) -> bool
  # denominator is the direction's component along the face normal;
  # numerator is how far the origin is beyond the face.
  IF denominator > 0 THEN
    IF numerator > denominator * leave THEN RETURN false   # enters after it leaves
    IF numerator > denominator * enter THEN enter = numerator / denominator
    RETURN true
  IF denominator < 0 THEN
    IF numerator > denominator * enter THEN RETURN false
    IF numerator > denominator * leave THEN leave = numerator / denominator
    RETURN true
  # Parallel to this face: the ray never crosses it, so the box is either
  # entirely in front of the ray's plane or entirely behind it.
  RETURN numerator <= 0
```

**Invariants** — The comparisons are written as products rather than as divisions, so the
division happens only when the interval actually moves. That is the reason the routine reads
the way it does, and it also means the sign of the denominator has to select between two
mirrored comparison orders — multiplying an inequality by a negative number flips it. A
rebuild that divides first may write one comparison instead of two, at the cost of a division
per face and an infinity to handle when the direction is parallel.

The six faces are passed as `(+d.a, -origin.a - half.a)` and `(-d.a, +origin.a - half.a)` for
each axis `a`, which is the half-extent form of "distance beyond the near face" and "distance
beyond the far face".

## Validity

**Contract** — Rotation, centre and half-extents all finite and normal.
