# src/xrCore/_vector2.h

> The 2-vector: screen positions, texture coordinates, and the horizontal plane the AI navigates in.

**Needs** — [`math_constants.h`](math_constants.h.md) · [`_std_extensions.h`](_std_extensions.h.md)
**Used by** — [`_fbox2.h`](_fbox2.h.md) · [`_matrix.h`](_matrix.h.md) · [`_plane2.h`](_plane2.h.md) · [`_rect.h`](_rect.h.md) · [`_stl_extensions.h`](_stl_extensions.h.md) · [`vector.h`](vector.h.md) · [`xr_ini.h`](xr_ini.h.md) · [`xr_input.h`](../xrEngine/xr_input.h.md)
**Tier floor** — T1: two consecutive components with no padding; it appears in vertex layouts as a texture coordinate pair and in on-disk data as a grid position.

## Purpose

Two components, templated over the component type and instantiated for 32-bit float and 32-bit signed integer. The float instantiation is texture coordinates, screen positions and horizontal-plane directions; the integer instantiation is grid coordinates and screen pixels.

It carries the same in-place/into-this calling convention as [`_vector3d.h`](_vector3d.h.md) and for the same reason: no temporaries.

## State

```text
RECORD Vector2 of T
  x : T
  y : T
```

Exactly two components, no padding. Indexed access is defined and aliases the named fields.

## Operations

| Group | Operations |
|---|---|
| assignment | from a pair of floats, doubles or integers, or from another 2-vector |
| arithmetic | add, subtract, multiply, divide — against a vector or a scalar, in place or into this |
| componentwise | minimum, maximum, absolute value |
| rotation | rotate a quarter turn; take the perpendicular, both as an in-place operation and as a returned value |
| products | dot product; a scalar cross product (the signed area of the parallelogram) |
| length | squared magnitude, magnitude, distance to another point |
| normalization | normalize, and a safe form that leaves a zero vector alone; each in place or from another |
| interpolation | arithmetic mean of two; geometric mean of two |
| multiply-add | position plus direction times scalar |
| similarity | componentwise within an epsilon, with a shared or per-component tolerance |
| angle | heading from the vector |

## Notes

**The quarter-turn rotation and the perpendicular disagree about direction.** Rotating gives `(y, -x)`; the perpendicular operation named "cross" gives `(D.y, -D.x)` — the same — but the scalar cross product is `y * p.x - x * p.y`, which is the *negative* of the usual convention `x * p.y - y * p.x`. Any rebuild using these to decide which side of a line a point lies on must check the sign against a known case rather than assuming.

**The heading conversion measures from the negative-`y` axis, clockwise**, matching the 3-vector's heading convention in the horizontal plane. It handles the zero-`y` cases explicitly and returns zero for the zero vector rather than failing. This is the same angle convention as everywhere else in the engine, and getting it wrong rotates the whole world by a quarter turn.

**The geometric mean takes a square root of a product of components**, which is only meaningful for positive components — it is used for blending attenuation factors, not positions.

The indexed accessor is declared such that it can be used on a constant vector to obtain a mutable reference, which is a defect rather than a decision; a rebuild has one accessor per constness.
