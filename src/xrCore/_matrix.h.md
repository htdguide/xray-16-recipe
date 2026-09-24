# src/xrCore/_matrix.h

> The 4×4 transform: sixteen reals in a fixed order, the conventions every transform in the engine obeys, and the operations short enough to inline — point transforms, projections, camera frames.

**Needs** — [`_vector3d.h`](_vector3d.h.md) · [`_vector2.h`](_vector2.h.md) · [`_vector4.h`](_vector4.h.md) · [`_quaternion.h`](_quaternion.h.md) · [`_std_extensions.h`](_std_extensions.h.md) · [`math_constants.h`](math_constants.h.md) · [`../utils/xrMiscMath/matrix.cpp`](../utils/xrMiscMath/matrix.cpp.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)

**Used by** — [`matrix.cpp`](../utils/xrMiscMath/matrix.cpp.md) · [`quaternion.cpp`](../utils/xrMiscMath/quaternion.cpp.md) · [`xr_collide_defs.h`](../xrCDB/xr_collide_defs.h.md) · [`BoneEditor.cpp`](Animation/BoneEditor.cpp.md) · [`NET_utils.cpp`](NET_utils.cpp.md) · [`_fbox.h`](_fbox.h.md) · [`_math.cpp`](_math.cpp.md) · [`_matrix33.h`](_matrix33.h.md) · [`_obb.h`](_obb.h.md) · [`_plane.h`](_plane.h.md) · [`_quaternion.h`](_quaternion.h.md) · [`dump_string.cpp`](dump_string.cpp.md) · [`vector.h`](vector.h.md) · [`xrCore.h`](xrCore.h.md) · _and 1 more_

**Tier floor** — T1: the sixteen reals are handed to the graphics driver as a constant buffer and appear in shipped model and level files as a memory image, so the order and the absence of padding are part of the contract.

## Purpose

Every placement in the engine is one of these: bone poses, object transforms, the view and
projection matrices, particle frames, the transform that maps the unit cube onto a bounding
box. This header defines the storage and the conventions; the algorithms too long to inline
— composition, inversion, the rotation constructors, the Euler conversions — live in
[`matrix.cpp`](../utils/xrMiscMath/matrix.cpp.md), and that split is a size decision, not a
design one.

The conventions below are the load-bearing content of this page. They are not derivable from
the code, they cannot be changed independently of the shipped data, and getting any one of
them wrong produces a world that is consistently and subtly mirrored, rotated or inside out
in a way no assertion will catch.

## The conventions

1. **Row vectors.** A point is a row and is multiplied on the *left*: `p' = p × M`. The
   consequence is that the basis vectors are the **rows**, not the columns.
2. **Translation lives in the fourth row.** The fourth column is `(0, 0, 0, 1)` for every
   transform that is not a projection.
3. **`mul(A, B)` applies B first, then A.** The argument order is reversed relative to the
   underlying matrix product. This is stated here because it is where rebuilds go wrong.
4. **A positive angle rotates clockwise** when looking along the axis. This is the opposite
   of the convention most readers carry, and every authored angle in the game data assumes it.
5. **The Euler rotation sequence is Z, then X, then Y** — bank, then pitch, then heading.
6. **The first three basis rows are named** right, up and forward (`i`, `j`, `k`), and the
   fourth is the position (`c`). Elsewhere in the engine the same four are called R, N, D and
   T — right, normal, direction, translation. Two vocabularies for one thing; a rebuild picks
   one.

## State

```text
RECORD Transform
  row1 : (real, real, real)    right / R          # basis
  w1   : real                  0 for an affine transform
  row2 : (real, real, real)    up / N
  w2   : real                  0
  row3 : (real, real, real)    forward / D
  w3   : real                  0
  row4 : (real, real, real)    position / T
  w4   : real                  1
```

Sixteen reals, row by row, no padding, no tag. The same storage is addressed three ways —
by element name, as four named basis vectors with their homogeneous components, and as a 4×4
array — because different call sites want different views of one set of bytes. A rebuild
exposes whichever views its tier supports and must keep the *order*.

**Invariants**

- The fourth column is `(0, 0, 0, 1)` for every transform except a projection and anything
  derived from one. The affine composition and the affine inverse both assume it without
  checking, and they are the forms used for bones and objects — that is, for almost
  everything.
- Nothing in the type enforces orthonormality of the basis rows. Transforms carrying scale
  and shear are legal and appear (a bounding box's transform is a pure scale), which is why
  the inverse is a general linear inverse rather than a transpose.
- The global **identity transform** is a mutable value filled during startup rather than a
  compile-time constant — see [`_math.cpp`](_math.cpp.md). Everything reads it; nothing is
  supposed to write it.

## Composition

**Contract** — Four in-place shapes over the two out-of-place products in
[`matrix.cpp`](../utils/xrMiscMath/matrix.cpp.md): compose before or after this transform,
with or without the projection column. Each copies this transform out of the way, because
the underlying product may not alias its destination — that copy is the entire reason the
in-place forms exist as separate names.

**Notes** — The affine form is the one used for per-bone and per-object work and skips a
quarter of the arithmetic. Using it on a projection matrix silently discards the projective
terms.

## Inversion and transposition

**Contract** — Three inverses and a transpose, all implemented in
[`matrix.cpp`](../utils/xrMiscMath/matrix.cpp.md): the affine inverse (asserting or
reporting failure), the full inverse for projection matrices, and the transpose. Each has a
convenience in-place shape that copies first.

## Translation, scale and mirroring

**Contract** — Three families, distinguished by a suffix that is the most useful naming
convention in the type and should survive a rebuild:

| Suffix | Meaning |
|---|---|
| *(none)* | **set** the whole transform to identity and then apply this one thing |
| `_over` | **overwrite** only the affected entries, leaving the rest of the transform alone |
| `_add` | **combine** with what is already there |

So the plain translate builds a pure translation matrix; translate-over replaces the position
row of an existing transform; translate-add shifts it. Mirroring exists in all three shapes
about each of the three axes — a mirror flips the sign of one diagonal entry, which reverses
handedness and therefore triangle winding downstream. Scale exists only in the "set" shape.

## `build_projection` and `build_projection_HAT`

**Contract** — Build a perspective projection from a vertical field of view (or, in the
second form, from the tangent of its half-angle, which is what callers that vary the field of
view per frame actually hold), an aspect ratio, and the near and far distances. Asserts that
the near and far planes are distinct and the half-angle tangent is non-zero.

```text
FUNCTION perspective(half_fov_tangent, aspect, near, far) -> Transform
  cot = 1 / half_fov_tangent
  depth_scale = far / (far - near)
  RETURN rows
    ( aspect * cot,  0,    0,                        0 )
    ( 0,             cot,  0,                        0 )
    ( 0,             0,    depth_scale,              1 )
    ( 0,             0,    -depth_scale * near,      0 )
```

**Invariants** — Depth maps to the range zero at the near plane through one at the far
plane, and the homogeneous divisor is taken from the **third** row's fourth entry — that is,
the view-space forward distance becomes the divisor. This is the convention of the graphics
interface the engine was written against; a rebuild targeting an interface whose depth range
is minus-one to one, or whose handedness differs, must convert here and only here, because
every shipped shader's depth arithmetic assumes this output.

**Notes** — The aspect ratio multiplies the *horizontal* term, so the field of view given is
vertical. Which of the two the parameter means is not recoverable from the name and is the
kind of thing a rebuild discovers by rendering a distorted frame.

## `build_projection_ortho`

**Contract** — Orthographic projection from a width, a height and the two depth distances.
Depth maps to zero at the near plane and one at the far plane, matching the perspective form
so the two are interchangeable downstream. Used for shadow-map cascades and for
screen-aligned drawing.

## `build_camera` and `build_camera_dir`

**Contract** — Build a view transform from an eye position and either a look-at point or a
look direction, plus a world up vector. Allocation-free, total apart from the degenerate
inputs below.

```text
FUNCTION view_from(eye, forward, world_up) -> Transform
  # Make the up vector perpendicular to the forward direction by removing
  # the component along it -- the caller supplies a rough up (usually world
  # vertical) and gets a proper orthonormal frame back.
  up = normalize(world_up - forward * dot(world_up, forward))
  right = cross(up, forward)

  # A view transform is the inverse of the camera's placement. For an
  # orthonormal basis the inverse of the rotation is its transpose, which is
  # why the basis vectors are written down the COLUMNS here and across the
  # rows everywhere else in the engine.
  rows 1..3 = the transpose of (right, up, forward)
  row 4 = ( -dot(eye, right), -dot(eye, up), -dot(eye, forward) )
```

**Invariants** — The look-at form normalizes the computed forward direction; the direction
form does **not** — it trusts the caller's direction to be unit already. Feeding a non-unit
direction to the second yields a view transform with baked-in scale.

Both fail when the forward direction is parallel to the world up: the projected up vector
becomes zero and normalizing it produces garbage. Nothing checks. Every camera in the engine
avoids it by clamping pitch short of vertical, which is why the check is absent — and is
exactly the kind of implicit contract a rebuild must re-establish deliberately.

## `inertion` — transform smoothing

**Contract** — Blend every one of the sixteen entries towards another transform's, by a
factor. Used to make a camera or an attached object lag its target instead of snapping.

**Notes** — This is an elementwise blend, not a proper interpolation of rotations: blending
two rotations this way passes through matrices that are not rotations at all, shrinking
towards the midpoint as the angle between them grows. It is correct-looking for the small
per-frame angles it is used for and visibly wrong for large ones. A rebuild that wants the
same effect for large angles decomposes to a quaternion and a translation, interpolates each
properly, and recomposes — which is what the animation layer already does, and is why this
operation is used only for cameras.

The blend factor's sense is inverted from the obvious reading: the factor weights *this*
transform and its complement weights the argument, so a factor of one means "do not move".

## The point transforms

**Contract** — Six shapes, all writing into a destination or operating in place, all
allocation-free. They differ in what they do with the homogeneous coordinate, and choosing
wrongly is a common and silent error:

| Operation | Divides by w | Applies translation | Use |
|---|---|---|---|
| transform a point, affine | no | yes | positions under an affine transform — the common case |
| transform a direction | no | **no** | normals, axes, velocities |
| transform a point, projective | yes | yes | positions through a projection |
| transform to a 4-component result | keeps w | yes | when the caller wants the divisor |
| 3-in, 2-out | no | yes | projecting to a plane, dropping the third coordinate |
| 2-in, 3-out | no | yes | lifting a planar point into space |

**Invariants** — The direction form omits the translation row. This is the distinction
between transforming a point and transforming a vector, and it is the one the type makes
explicit by giving each its own name rather than by carrying a homogeneous component through.
That is a good decision and a rebuild should keep it.

The affine point transform is the one marked "preferred" throughout, because the projective
divide costs a division per point and is unnecessary for every transform whose fourth column
is `(0, 0, 0, 1)` — which is all of them except the projection.

## The Euler-angle accessors

**Contract** — Set or read the rotation as three angles. Two naming schemes over one
underlying pair implemented in [`matrix.cpp`](../utils/xrMiscMath/matrix.cpp.md):

- **heading / pitch / bank** — the engine's own names, and the order the underlying routine
  takes them in.
- **x / y / z** — the same three angles reordered so that x is pitch, y is heading and z is
  bank. This is the order the *script layer and the game data* use, which is why it exists.
- **an inverted x/y/z form** — the same, with all three angles negated, for the call sites
  that hold angles in the opposite sense.

**Invariants** — The mapping between the two schemes is fixed: `x` is pitch, `y` is heading,
`z` is bank. Every authored orientation in the game data is three numbers in the x/y/z order
and is fed through this mapping, so the mapping is as frozen as the rotation sequence itself.

**Notes** — The existence of the negated variants is a symptom rather than a design: two
parts of the engine disagreed about the sign of an angle and the disagreement was resolved by
providing both. A rebuild should pick one sense, convert at the data boundary, and delete the
other — but must first determine which sense each *caller* wanted, because the code cannot
tell you.

## Validity

**Contract** — A transform is valid when all sixteen entries are finite and normal. Used in
assertions throughout the animation and physics layers, where a degenerate bone or a solver
blow-up shows up here first.
