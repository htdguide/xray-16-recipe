# src/xrCore/_vector3d_ext.h

> Value-returning vector arithmetic: the operator style, for code where readability beats the no-temporaries rule.

**Needs** — [`_vector3d.h`](_vector3d.h.md)
**Used by** — [`vector3d_ext.cpp`](../utils/xrMiscMath/vector3d_ext.cpp.md)
**Tier floor** — T2: it returns vectors by value; nothing here constrains layout beyond what the underlying type already fixes.

## Purpose

The engine's math vocabulary is written to avoid temporaries — every operation writes into an existing vector (see [`_vector3d.h`](_vector3d.h.md)). That is right in the frame loop and wrong everywhere else, where an expression reads better than four statements. This file is the escape hatch: the same operations spelled as value-returning functions and operators.

Which style a call site uses is a judgement about whether it is on a hot path. A rebuild on a tier where values are free should use this style everywhere and delete the other.

## Exported units

- **Construction** — a vector with all three components equal; a vector from three components; a vector from a heading/pitch pair.
- **Operators** — addition, subtraction, negation, multiplication by a scalar on either side, division by a scalar.
- **Componentwise** — minimum, maximum, absolute value, all returning new vectors.
- **Normalization** — returns a normalized copy.
- **Length** — magnitude and squared magnitude as free functions.
- **Products** — dot product and cross product as free functions returning values.
- **Angles** — a direction from a heading/pitch pair; the angle between two directions.
- **Planar rotation** — rotate a point by an angle. The plane it rotates in is not named by the signature and must be checked at the definition before use.

## Notes

Division by a scalar computes the reciprocal once and multiplies, which is a different result from dividing each component — the difference is one rounding and it is visible in the physics layer's determinism. A rebuild reproducing recorded trajectories must match it.

The name of the squared-magnitude helper is misspelled in the source (`sqaure_magnitude`). Carrying the typo is pointless; carrying the knowledge that it exists in the original is not, because a search for the correct spelling will miss its call sites.
