# src/utils/xrMiscMath/quaternion.cpp

> Extracts a rotation quaternion from a transform, robustly, including the cases where the
> obvious formula loses all its precision.

**Needs** — [`xrCore/_quaternion.h`](../../xrCore/_quaternion.h.md) · [`xrCore/_matrix.h`](../../xrCore/_matrix.h.md) · [`xrCommon/math_funcs_inline.h`](../../xrCommon/math_funcs_inline.h.md)
**Used by** — [`_quaternion.h`](../../xrCore/_quaternion.h.md)
**Tier floor** — T2: pure arithmetic, held below T3 only because it runs per bone per frame in the animation blend

## Purpose

The quaternion type is otherwise entirely inline; this file holds the one operation that
is too long and too conditional to inline — conversion *from* a transform. The reverse
direction lives in [`matrix.cpp`](matrix.cpp.md), which is the split a rebuilder should
question: the two halves of one conversion sit in two files because of which type each
belongs to, not because of anything about the algorithm.

## State

`Stateless.`

## `set(transform)` — transform to quaternion

**Contract** — Reads the upper-left 3×3 rotation block of a transform and writes the
equivalent unit quaternion. The translation and projection parts are ignored. The
transform must be a pure rotation: any embedded scale or shear is not detected and
produces a silently wrong result. Total in the sense that it never fails loudly — see the
invariant below for what happens when it fails quietly.

**Invariants** — There is one case where the quaternion is **left completely unwritten**:
if none of the four extraction routes produces a usable scale factor, the routine returns
with the destination holding whatever it held before. A rebuild should either reproduce
that (and accept that callers must pre-seed the destination) or, better, write identity
and say so — but it must know that the original signals nothing here.

```text
FUNCTION quaternion_from_rotation(m) -> quaternion
  trace = m[1][1] + m[2][2] + m[3][3]
  IF trace > 0
    # the scalar part is the largest component; extract it first and read the
    # three vector components from the antisymmetric off-diagonal differences
    s = sqrt(trace + 1)
    w = s / 2
    s = 1 / (2*s)
    x = (m[3][2] - m[2][3]) * s
    y = (m[1][3] - m[3][1]) * s
    z = (m[2][1] - m[1][2]) * s
    RETURN (w, x, y, z)

  # trace <= 0 means the scalar part is small, so dividing by it would amplify
  # rounding. Extract whichever vector component is largest instead.
  candidates = the three axes, ordered with the largest diagonal entry first
  FOR EACH axis IN candidates
    s = sqrt( m[axis][axis] - (sum of the other two diagonal entries) + 1 )
    IF s > EXTRACTION_TOLERANCE
      component[axis] = s / 2
      s = 1 / (2*s)
      w                = (antisymmetric pair for this axis) * s
      other components = (symmetric pairs for this axis) * s
      RETURN the quaternion
  # every route was degenerate: return with the destination unmodified
```

**Notes**

*Why four routes.* A rotation quaternion has four components and exactly one of them is
guaranteed to be at least a half in magnitude. Each extraction route recovers one
component directly from a square root and derives the other three by dividing by it. Take
the route whose component is largest and the division is well conditioned; take the wrong
route and the division amplifies rounding without bound. The trace test picks the scalar
route; the diagonal comparison picks among the three vector routes.

*The tolerance.* The guard on each route is a fixed one tenth. It is not an epsilon — it is
a generous floor chosen so that the divisor is comfortably away from zero rather than
merely non-zero. Any route rejected by it is retried through the next candidate, so the
constant trades a little accuracy in route selection for certainty that no division ever
runs near the cliff. A rebuild can pick a different value; it cannot drop the guard.

*The selection is imperfect and that is deliberate.* The comparison that ranks the three
diagonal entries tests the third against the *first* in both branches, never against the
second. As a result a rotation whose second diagonal entry is the largest can still be
routed to the third axis. The fallback chain is what makes this harmless: every route that
fails its tolerance falls through to the next, and the chain is ordered differently for
each starting choice so that all three are eventually tried. A rebuild is free to rank the
three correctly — the outputs agree wherever both are well conditioned — but must keep the
fallback chain, because the ranking is not the safety net, the chain is.

*Dead code carried in the file.* A disabled routine recovers an angular velocity from two
quaternions and a time step, for the physics layer. Its only call site is itself commented
out. It is not part of the module's surface and a rebuild should not carry it.
