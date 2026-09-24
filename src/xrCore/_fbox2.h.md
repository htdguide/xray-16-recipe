# src/xrCore/_fbox2.h

> The two-dimensional axis-aligned box: the same vocabulary as the 3D box, one axis shorter, used for texture atlas regions, UI extents and the horizontal footprint of things.

**Needs** — [`_vector2.h`](_vector2.h.md) · [`_std_extensions.h`](_std_extensions.h.md) · [`../xrCommon/math_funcs_inline.h`](../xrCommon/math_funcs_inline.h.md)

**Used by** — [`r__sector.h`](../Layers/xrRender/r__sector.h.md) · [`level_graph_inline.h`](../xrAICore/Navigation/level_graph_inline.h.md) · [`level_graph_vertex.cpp`](../xrAICore/Navigation/level_graph_vertex.cpp.md) · [`vector.h`](vector.h.md)

**Tier floor** — T1: four consecutive reals, passed by value, and shaped like the 3D box so the two can be read by the same loader code.

## Purpose

The 2D twin of [`_fbox.h`](_fbox.h.md). It exists because several problems are genuinely
planar — a lightmap's region in an atlas, a level's footprint seen from above, a detail-object
patch's extent — and doing them with a 3D box means carrying a meaningless third axis through
every comparison.

Note that the engine *also* has a rectangle type, [`_rect.h`](_rect.h.md), with a nearly
identical field layout and an overlapping but different operation set. The two are not
unified, and the split is not principled: this one is named and shaped like a bounding volume
and is used for accumulation, while the rectangle is named and shaped like a screen region
and is used for clipping. A rebuild should have one type and should decide which vocabulary
it keeps.

## State

```text
RECORD Box2
  minimum : (real, real)
  maximum : (real, real)
```

Four reals in that order. The invariants are the 3D box's, one axis shorter: the **invalid**
box is minimum at the largest representable real and maximum at the smallest, and is the
identity for accumulation; the **identity** box is the unit square centred on the origin,
spanning -0.5 to +0.5; the zero box is neither.

## The shared vocabulary

Everything here behaves exactly as the matching operation on
[`_fbox.h`](_fbox.h.md#growth-and-shift) does, in two dimensions:

| Group | Operations |
|---|---|
| construction | from two corners, from four components, from another box, to zero, to the unit square, to the accumulation identity |
| growth | grow and shrink, by a scalar or per-axis |
| shift | add and offset — again two names for one translation |
| accumulation | extend by a point; merge another box in; merge two into this one, invalidating first |
| predicates | contains a point or a box, inclusive at the boundary; overlap against another box; corners similar within the loose epsilon |
| measurement | size, half-size, circumradius, centre, circumscribed circle |

## `sort` — repair an inverted box

**Contract** — Swap the components of the two corners wherever the minimum exceeds the
maximum, producing a valid box from a pair of arbitrary points. Allocation-free.

**Notes** — This has no 3D counterpart, and it is the operation that makes the type usable
for *drag-selection* — a user drags from an arbitrary corner to another, and the result is
two points in no particular order rather than a minimum and a maximum. A rebuild that offers
a "from two arbitrary points" constructor makes this redundant.

## `Pick` and `pick_exact` — line against the box

**Contract** — Both report whether an infinite line through an origin passes through the box,
with no distance and no sign test, so a hit behind the origin counts. The difference is the
tolerance at the box's edge: the plain form is exact, the other admits a hit within the loose
epsilon of the boundary.

```text
FUNCTION line_hits_box(origin, direction) -> bool
  low  = minimum - origin
  high = maximum - origin
  FOR EACH axis a WITH direction.a not near zero
    FOR EACH edge IN (low.a, high.a)
      t = edge / direction.a
      IF the other coordinate of t*direction lies between low and high
        RETURN true          # widened by epsilon on both ends in the lenient form
  RETURN false
```

**Notes** — The two differ in one more respect that is easy to miss: the exact form skips an
axis when the direction component is within epsilon of zero, while the lenient form skips it
only when the component is *exactly* zero. So the lenient form divides by very small numbers
that the exact form declines to divide by. Both then rely on the resulting huge parameter
failing the range test, which it does. A rebuild picks one convention.

## `Pick2` — the ray hit with its point

**Contract** — The candidate-plane test, identical in structure to the 3D box's
[`Pick2`](_fbox.h.md#pick2--the-ray-hit-with-its-point) with one axis fewer: find the last
plane crossed, reject a negative parameter, confirm the remaining coordinate lands inside the
edge. Returns a boolean rather than the three-way classification the 3D version returns —
"origin inside" and "hit from outside" are both reported as true, and the caller cannot tell
them apart. That divergence between the 2D and 3D forms is an inconsistency, not a decision.

Carries the same hazard as the 3D form: coordinates of axes the origin is already inside on
are not written, so the caller's point must be initialized.

## `getpoint` and `getpoints` — the corners

**Contract** — Meant to yield the four corners of the box, by index or all at once.

**Notes** — **These two are broken in the original and a rebuild must not copy them.** Both
produce `(min.x, min.y)` twice and `(max.x, min.y)` twice; the maximum's y component is never
read, so the two corners on the far edge are never produced. The duplication pattern suggests
the 3D version's eight-corner table was edited down by deleting the z component and the y
component was left out of the edit. Nothing in the engine calls either — which is why the bug
has survived — so there is no behaviour to preserve. The correct four corners, keeping the
3D type's ring ordering, are `(min.x, min.y)`, `(min.x, max.y)`, `(max.x, max.y)`,
`(max.x, min.y)`.
