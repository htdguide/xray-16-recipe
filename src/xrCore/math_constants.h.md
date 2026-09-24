# src/xrCore/math_constants.h

> The three comparison epsilons and the circle constants, as single-precision values.

**Needs** — _(none)_
**Used by** — [`vector.cpp`](../utils/xrMiscMath/vector.cpp.md) · [`math_funcs.h`](../xrCommon/math_funcs.h.md) · [`math_funcs_inline.h`](../xrCommon/math_funcs_inline.h.md) · [`_bitwise.h`](_bitwise.h.md) · [`_color.h`](_color.h.md) · [`_matrix.h`](_matrix.h.md) · [`_plane.h`](_plane.h.md) · [`_plane2.h`](_plane2.h.md) · [`_quaternion.h`](_quaternion.h.md) · [`_sphere.h`](_sphere.h.md) · [`_vector2.h`](_vector2.h.md) · [`_vector3d.h`](_vector3d.h.md) · [`_vector4.h`](_vector4.h.md) · [`vector.h`](vector.h.md)
**Tier floor** — T3: named constants.

## Purpose

Two families of constant, and the first is the interesting one.

## The comparison epsilons

**Contract** — Three tolerances, used as the default in every "are these similar" test in the engine:

| Name | Value | Used for |
|---|---|---|
| strict | 0.0000001 | near the limit of single precision; "is this exactly zero" |
| standard | 0.0000100 | the default for scalar comparison |
| loose | 0.0010000 | the default for vector, matrix and quaternion similarity |

**Invariants** — The loose value is the default on the *aggregate* similarity tests and the standard value on scalar ones. That gap of two orders of magnitude is deliberate: a position that is within a millimetre is the same position for gameplay purposes, and tightening it makes animation blending and physics resting states chatter.

**Notes** — These are absolute tolerances, not relative ones. At the scale the game's worlds are authored in — a level is hundreds of metres across in units of one metre — an absolute millimetre is meaningful. A rebuild that changes the world scale must rescale these with it, and a rebuild that switches to relative comparison changes resting-contact behaviour in the physics layer.

## Circle constants

**Contract** — Pi, and pi multiplied and divided by 2, 3, 4, 6 and 8, plus the reciprocal square root of two. All single-precision.

**Notes** — They are written with more decimal digits than a single-precision float can hold, because they were once available as double-precision macros too. The extra digits are harmless and carry no meaning.

The file begins by removing any pre-existing definitions of these names from the platform's own headers, which is incidental: the engine wants its own single-precision values and will not accept the platform's double-precision ones silently.
