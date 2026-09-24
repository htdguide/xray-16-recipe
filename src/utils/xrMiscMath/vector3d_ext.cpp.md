# src/utils/xrMiscMath/vector3d_ext.cpp

> The handful of value-returning vector helpers that could not be written inline.

**Needs** — [`xrCore/_vector3d_ext.h`](../../xrCore/_vector3d_ext.h.md) · [`xrCore/_vector3d.h`](../../xrCore/_vector3d.h.md) · [`xrCommon/math_funcs_inline.h`](../../xrCommon/math_funcs_inline.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3 in isolation — scalar arithmetic with no layout or device contact; T2 in practice because every call site is inside the frame loop and these return aggregates by value

## Purpose

The 3-vector's own operations all write into a destination the caller already owns, which
reads badly in expressions. This file is the value-returning face: the same geometry
expressed as functions that take operands and return a fresh vector, so that vector
algebra can be written as algebra. Most of that face is inline in the header; the five
routines here are out of line for no reason a reader can recover — they are neither
longer nor heavier than the inline ones. A rebuild should treat the whole value-returning
surface as one thing and ignore the split.

## State

`Stateless.`

## `dotproduct` and `crossproduct`

**Contract** — The two products, taking both operands and returning the result. Total,
no aliasing concerns since the result is fresh.

## `cr_vectorHP`

**Contract** — The unit direction for a heading/pitch pair, returned rather than written.
Identical in meaning to the vector's own heading/pitch setter; see
[`vector.cpp`](vector.cpp.md#3-vector--heading-and-pitch) for the convention, which is the
authority.

**Notes** — The formula is written out a second time here instead of delegating. A rebuild
must keep the two in step or, better, have only one.

## `angle_between_vectors`

**Contract** — The unsigned angle between two directions, in `[0, π]`. Returns zero — not
an error — when either input is shorter than a millionth, so a caller holding a degenerate
direction gets "no rotation" rather than a NaN.

```text
FUNCTION angle_between(v1, v2) -> real
  IF magnitude(v1) < TINY OR magnitude(v2) < TINY
    RETURN 0
  c = dot(v1, v2) / (magnitude(v1) * magnitude(v2))
  # rounding in the division can push the cosine a hair outside its own domain;
  # arc-cosine would return not-a-number, so clamp before asking
  c = clamp(c, -1, +1)
  RETURN arccos(c)
```

**Notes** — The clamp is the load-bearing line. Two vectors that are numerically identical
routinely produce a cosine of 1.0000001 in single precision, and every rebuild that omits
the clamp discovers this the first time something looks at itself.

## `rotate_point`

**Contract** — Rotates a point about the vertical axis by an angle and returns it **with
the vertical component set to zero**. It is a ground-plane rotation, not a general one,
despite the name.

**Notes** — Discarding the vertical component rather than preserving it is the surprise. It
is correct for the callers, which are working in the horizontal plane, but a rebuild that
makes the name honest — by preserving the height, or by renaming — will change behaviour
at any site that was relying on the flattening.
