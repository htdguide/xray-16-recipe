# src/xrCommon/math_funcs_inline.h

> The engine's three tolerances and the rules for using them, plus clamping, degree conversion and grid snapping.

**Needs** — [`xrCore/math_constants.h`](../xrCore/math_constants.h.md) · [`xrCore/_bitwise.h`](../xrCore/_bitwise.h.md)
**Used by** — [`matrix.cpp`](../utils/xrMiscMath/matrix.cpp.md) · [`pch.hpp`](../utils/xrMiscMath/pch.hpp.md) · [`quaternion.cpp`](../utils/xrMiscMath/quaternion.cpp.md) · [`vector.cpp`](../utils/xrMiscMath/vector.cpp.md) · [`vector3d_ext.cpp`](../utils/xrMiscMath/vector3d_ext.cpp.md) · [`xrMiscMath.cpp`](../utils/xrMiscMath/xrMiscMath.cpp.md) · [`math_funcs.h`](math_funcs.h.md) · [`_compressed_normal.cpp`](../xrCore/_compressed_normal.cpp.md) · [`_cylinder.cpp`](../xrCore/_cylinder.cpp.md) · [`_fbox.h`](../xrCore/_fbox.h.md) · [`_fbox2.h`](../xrCore/_fbox2.h.md) · [`_matrix33.h`](../xrCore/_matrix33.h.md) · [`_plane.h`](../xrCore/_plane.h.md) · [`_plane2.h`](../xrCore/_plane2.h.md) · _and 5 more_
**Tier floor** — T3: scalar arithmetic and three chosen constants.

## Purpose

Nothing in this engine compares two real numbers for equality. It compares them for
*similarity*, against one of three fixed tolerances, and which tolerance a call site uses
is a decision about what that site considers "the same". This file names the comparisons
and fixes their defaults. A rebuild deletes the arithmetic wrappers and keeps the
tolerances, because the tolerances are tuned against the shipped data — level geometry
authored at a particular scale, in metres.

## The three tolerances

```text
EPS_TINY   = 0.0000001     # "is this exactly zero"     - default for is-zero tests
EPS        = 0.0000100     # "are these the same value" - default for similarity
EPS_LOOSE  = 0.0010000     # "same to within a millimetre at world scale"
```

**Invariants**

```text
# The tolerances are ABSOLUTE, not relative.
#   Two large coordinates that differ in their last bits are NOT similar under
#   these tests, and two tiny values are similar even when they differ by orders
#   of magnitude. This is correct for the engine's use: the numbers being compared
#   are world coordinates in metres, direction components in [-1,1], and dot
#   products, all of which live in a known range. A rebuild that substitutes a
#   relative comparison changes which geometry welds, which portals close, and
#   which contacts are discarded.

# The loose tolerance is the world-space one.
#   Distances in this engine are metres. A millimetre is below what the level
#   geometry was authored to and below what the physics solver resolves, so the
#   loose tolerance is the "same place" test. It is the one to reach for when
#   comparing positions; the tighter two are for normalized and derived values.
```

## `fsimilar` / `dsimilar` — are two values the same

**Contract** — true when the magnitude of the difference is strictly below the tolerance,
defaulting to the middle tolerance. Strictly below, so a difference of exactly the
tolerance is *not* similar. Two variants exist only because the physics solver works in
higher precision; the meaning is identical.

## `fis_zero` / `dis_zero` — is this value zero

**Contract** — true when the magnitude is strictly below the tolerance, defaulting to the
tiniest one. Note the different default from the similarity test: asking "is this zero" is
a stricter question in this engine than asking "are these the same", by two orders of
magnitude. This asymmetry is deliberate and load-bearing — a vector length is tested for
zero before a division, where a false positive costs a divide-by-zero and a false negative
costs nothing.

## `deg2rad` / `rad2deg`

**Contract** — convert between degrees and radians. Radians are the engine's working unit
everywhere; degrees appear only at the two boundaries where humans wrote the number — the
configuration files and the console — so these conversions belong at the parse site and
nowhere else.

**Notes** — the half-turn constant used here is given to more decimal places than the
working precision can hold. That is not an error: the same constants are also consumed at
higher precision by the physics solver, so they are written to full precision once and
narrowed by whoever reads them.

## `clamp` — constrain in place

**Contract** — force a value into a range, modifying it. Below the low bound it becomes the
low bound; above the high bound, the high bound.

**Invariants**

```text
# The low bound is tested first, so an INVERTED range (low above high) yields the
# low bound, silently. Nothing validates. A rebuild that asserts here will fire on
# configuration-driven ranges that nobody noticed were inverted.
```

## `clampr` — constrain and return

**Contract** — the same constraint, returning the constrained value instead of modifying
the argument. Same inverted-range behaviour. Two forms exist purely because C++ makes
in-place and by-value awkward to unify; a rebuild needs one.

## `snapto` — quantize to a grid

**Contract** — round a value to the nearest multiple of a given grid step. A step of zero
or less means "no grid" and returns the value unchanged, which is how callers disable
snapping without a branch.

```text
FUNCTION snapto(value, step) -> real
  IF step <= 0
    RETURN value                        # the documented "snapping off" input
  RETURN floor((value + step / 2) / step) * step
```

**Invariants**

```text
# Ties round toward positive infinity, not away from zero and not to even.
#   Half a step above a grid line goes up; half a step below goes up as well.
#   Consistent across the sign change, which matters because this is used to align
#   editor-placed objects and detail-object cells that straddle the origin.

# The intermediate quotient passes through a 32-bit signed integer, so the
# representable range is about two billion grid steps either side of zero. Beyond
# that the result is undefined. World coordinates are metres and levels are
# hundreds of metres across, so this is unreachable in practice.
```

## The single-argument arithmetic wrappers

**Contract** — absolute value, square root, sine and cosine, each in both working
precisions.

**Notes** — these exist only because the platform's own maths functions came in
precision-specific names that could not be selected by argument type, so every call site
would otherwise have had to spell the precision. They survive a rebuild as nothing at all:
use the language's own. They are named here only so that the twins for the rest of the
engine can refer to "the engine's absolute value" without a reader going looking for
something clever.
