# src/utils/xrMiscMath/xrMiscMath.cpp

> Normalizes a vector whose length the ordinary route cannot even represent.

**Needs** — [`xrCore/_vector3d.h`](../../xrCore/_vector3d.h.md) · [`xrCommon/math_funcs_inline.h`](../../xrCommon/math_funcs_inline.h.md) · [`vector.cpp`](vector.cpp.md) · [Platform assumptions](../../../SYSTEM-REQUIREMENTS.md#4-platform-assumptions) · [Seam: Rigid-body dynamics](../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: the whole routine is about the difference between two float widths and about what underflows in the narrower one, which a tier with a single opaque number type cannot express

## Purpose

The 3-vector's own normalization squares the components before it does anything else. For a
direction whose components are around a hundred-billionth, those squares underflow to zero
in single precision and the direction — which is perfectly well defined — is lost. This
file holds the routine that recovers it. It exists because the physics layer produces such
vectors routinely: contact normals and constraint axes shrink as a body settles, and a
solver handed a zero or a NaN instead of a unit axis explodes.

The file carries the module's name and one function. That is a naming accident, not a
structure; a rebuild puts this next to the other normalizations.

## State

`Stateless.`

## `exact_normalize`

**Contract** — Normalizes a 3-vector in place, working in the wider float width
throughout, and reports whether the input had a recoverable direction. Returns false only
for the exactly-zero vector, and in that case writes a fixed default unit vector so the
caller always ends up holding something of unit length. Total, allocation-free, no
diagnostics. Two entry points exist — one over a raw triple of components, one over the
vector record — because the physics layer hands over component storage it owns; they are
the same routine.

**Invariants** — On return the vector is unit length in the wider width, to within that
width's rounding, for every input except all-zero. There is no input for which the caller
receives a NaN or an infinity.

```text
FUNCTION exact_normalize(v) -> bool        # v is updated in place
  m2 = v.x*v.x + v.y*v.y + v.z*v.z         # computed wide, not narrow
  IF m2 > SAFE_SQUARE_THRESHOLD
    v = v scaled by (1 / sqrt(m2))         # ordinary route, and the common case
    RETURN true

  # The square underflowed, or came close. Divide through by the largest
  # component first: the scaled vector has one component of exactly 1 and two
  # of magnitude at most 1, so its squares are representable no matter how
  # small the original was.
  largest = the axis of v with the greatest magnitude
  IF magnitude of v along largest is zero
    v = (0, 1, 0)                          # the whole vector is zero; hand back
    RETURN false                           # a usable default rather than a fault
  a = the two components other than `largest`, each divided by that magnitude
  s = 1 / sqrt(a.first*a.first + a.second*a.second + 1)
  v.other_components = a scaled by s
  v.largest_component = s with the sign of the original component
  RETURN true
```

**Notes**

*Why the sign is transplanted rather than computed.* After dividing by the magnitude of
the largest component, that component is ±1 and its normalized value is exactly `s`; only
its sign is left to recover. Copying the sign across is not an optimization — it is the
only way to get it, since the magnitude used for the division threw the sign away.

*The threshold.* The fast-path test is against a hundred times the single-precision
rounding step — roughly `1.2e-5`. It is not tight: any squared magnitude comfortably above
the underflow floor would do. It is written as a single-precision literal and then used in
the wider width, which is harmless and is the sort of detail a rebuild should simply not
reproduce; what it must reproduce is a threshold that is (a) far above where the narrow
squares stop being representable and (b) far below any magnitude the callers actually
work with, so the slow path is genuinely rare.

*The default for the zero vector.* Straight up in the engine's axis convention. Nothing
derives it; it is "some unit vector", chosen so the caller's subsequent arithmetic stays
finite. A rebuild may pick a different one, but it must pick one — returning the zero
vector unchanged is what this routine exists to avoid.

*Precision.* Every intermediate is computed in the wider width and only the final
assignment narrows. That is the point of the routine and is not negotiable: doing the
arithmetic in the storage width reproduces exactly the failure being fixed. It also means
the reciprocal square root used here must be the exact one, not an approximation — see the
note in [`vector.cpp`](vector.cpp.md#rsqrt).

*Provenance.* The routine's shape — including its default vector and its threshold — comes
from the rigid-body library the engine is built against, so its behaviour is part of the
contract on that seam rather than a free choice.
