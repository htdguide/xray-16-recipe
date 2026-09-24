# src/xrCore/_vector3d.h

> The 3-vector: three floats, no padding, and the operation vocabulary the whole engine speaks in — positions, directions, normals, colours-as-vectors, Euler angle pairs.

**Needs** — [`xr_types.h`](xr_types.h.md) · [`math_constants.h`](math_constants.h.md) · [`_random.h`](_random.h.md) · [`utils/xrMiscMath/vector.cpp`](../utils/xrMiscMath/vector.cpp.md) · [`../xrCommon/math_funcs_inline.h`](../xrCommon/math_funcs_inline.h.md)
**Used by** — [`R_light.h`](../utils/xrLC_Light/R_light.h.md) · [`matrix.cpp`](../utils/xrMiscMath/matrix.cpp.md) · [`vector.cpp`](../utils/xrMiscMath/vector.cpp.md) · [`vector3d_ext.cpp`](../utils/xrMiscMath/vector3d_ext.cpp.md) · [`xrMiscMath.cpp`](../utils/xrMiscMath/xrMiscMath.cpp.md) · [`xrCDB.h`](../xrCDB/xrCDB.h.md) · [`xr_collide_defs.h`](../xrCDB/xr_collide_defs.h.md) · [`FMesh.hpp`](FMesh.hpp.md) · [`FS.h`](FS.h.md) · [`_compressed_normal.cpp`](_compressed_normal.cpp.md) · [`_cylinder.cpp`](_cylinder.cpp.md) · [`_cylinder.h`](_cylinder.h.md) · [`_fbox.h`](_fbox.h.md) · [`_matrix.h`](_matrix.h.md) · _and 15 more_
**Tier floor** — T1: it is exactly three consecutive floats with no header and no padding, because vertex buffers, physics structures and on-disk level data are all read and written as arrays of it.

## Purpose

This is the most-used type in the engine. Its layout is not an implementation detail: a vertex buffer handed to the graphics driver, a body's position handed to the rigid-body library, and a level file's vertex array are all arrays of this record, read in place. A rebuild may parse fields instead of mapping, but the byte layout must be reproducible.

The type is a template over the component type, instantiated for 32-bit float and 32-bit signed integer. The integer instantiation is used for grid and index triples, where most of the floating-point operations are meaningless but harmless.

## State

```text
RECORD Vector3 of T
  x : T
  y : T
  z : T
```

Exactly three components, in that order, no padding, no tag. Component access by index is defined and is the same as by name — several loaders iterate components.

**Invariants** — None are enforced by the type. A direction is expected to be unit length wherever an operation says so, and nothing checks; a position may be any finite value. The validity predicate (finite, not subnormal, not infinite) is the engine's standard assertion and is applied liberally in debug builds.

## The operation vocabulary

Every operation exists in two shapes: **in place** (this vector becomes the result) and **into this** (this vector becomes the result of operating on two others). Both return the vector so calls chain. That pairing is the whole calling convention of the engine's math layer, and it exists so that no temporary is ever created — which is the T1 constraint made visible in the interface.

| Group | Operations |
|---|---|
| assignment | set from components, from another vector of either precision, from a pointer to components |
| arithmetic | add, subtract, multiply, divide — each against a vector or a scalar, in place or into this |
| negation | invert, in place or from another |
| componentwise | minimum, maximum, absolute value |
| similarity | componentwise comparison within an epsilon, defaulting to the loose epsilon |
| length | squared magnitude, magnitude, set to a given length |
| normalization | normalize, normalize-safe (leaves a zero vector alone), each in place or from another; plus a form returning the magnitude it divided by |
| products | dot product, cross product |
| distance | to another point, squared, and the same restricted to the horizontal plane |
| interpolation | average of two, linear interpolation, and an *inertial* approach that moves a fraction of the way each call |
| multiply-add | position plus direction times scalar, in four argument shapes |
| barycentric | from three points and three weights, and from four points and four weights |
| normals | face normal from three points, normalized or not |
| angles | set from a heading/pitch pair, read back as a pair or individually |
| reflection | reflect a direction about a normal; slide a direction along a surface |
| bases | build an orthonormal basis around a direction, with and without assuming it is normalized |
| random | a random direction, a random direction within a cone, a random point in a box, a random point in a sphere |
| clamping | to a box, and the shorthand symmetric form |
| squeezing | snap components smaller than an epsilon to zero |
| alignment | snap to the nearest axis in the horizontal plane |

**Notes** — Heading and pitch are the engine's angle convention throughout: a direction is stored as two angles wherever an orientation must be authored or serialized compactly, and the conversion here is the definition. Roll is not part of it — a third angle appears only where a full orientation is needed.

The random operations take a generator by reference, defaulting to the engine's global one. That default is what makes physics reproducibility a property of *seeding the global generator*, and it is why the determinism criterion is stated as "the same level, input sequence and seed".

The alignment operation snaps to an axis *ignoring the vertical component*, which is what makes it useful for door hinges and cover directions and useless for anything else.

## Free functions

- **Validity** — all three components finite and normal.
- **Reciprocal square root** and **exact normalization** — declared here, defined in the math layer. The "exact" normalization reports whether the vector was long enough to normalize at all, which the physics layer needs because a zero-length normal is a solver blow-up rather than a rendering artefact.

## Notes

Almost every operation is declared here and defined in the math layer ([`utils/xrMiscMath/vector.cpp`](../utils/xrMiscMath/vector.cpp.md)), so this header is the vocabulary and not the algorithms. The few defined here — component access, set, the four arithmetic groups, componentwise minimum and maximum, absolute value, similarity and dot product — are the ones in the innermost loops.
