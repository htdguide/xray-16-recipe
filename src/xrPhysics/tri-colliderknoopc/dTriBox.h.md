# src/xrPhysics/tri-colliderknoopc/dTriBox.h

> The box's reach along a direction, and the point-in-box and segment-in-box tests its
> triangle case is built from.

**Needs** — [`dcTriListCollider.h`](dcTriListCollider.h.md) · [`TriPrimitiveCollideClassDef.h`](TriPrimitiveCollideClassDef.h.md) · [`../MathUtilsOde.h`](../MathUtilsOde.h.md) · [`../ode_include.h`](../ode_include.h.md) · [`dTriBox.cpp`](dTriBox.cpp.md)
**Used by** — [`dTriBox.cpp`](dTriBox.cpp.md) · [`dTriCollideK.h`](dTriCollideK.h.md)
**Tier floor** — T1: the box's payload layout is shared with the dynamics library, and these
routines are inlined into the collider's innermost loop.

## Purpose

Header-only helpers for [`dTriBox.cpp`](dTriBox.cpp.md), plus the box's record. They are here
rather than in the implementation because the collider's traversal needs the projection
inlined, and the rest travel with it.

## State

```text
RECORD Box
  side : three reals        # FULL side lengths, not half
```

**Invariants** — full lengths, halved at every use site. That is the dynamics library's
convention and the source of a steady stream of factor-of-two errors; a rebuild should store
half-extents and halve once, at load.

## box projection

**Contract** — how far a box reaches from its centre along a given direction: the sum over its
three local axes of the half-side times the absolute cosine between that axis and the
direction. Exact, and the only thing the mesh traversal asks of the box.

## `IsPtInBx` / `PointBoxTest`

**Contract** — is a point inside an oriented box, and if so how deep and out of which face?

```text
FUNCTION point_in_box(point, box) -> optional<(depth, normal)>
  bring both the point and the box centre into the box's frame
  FOR EACH local axis i
    depth[i] := half_side[i] - |offset along i|
    IF depth[i] < 0 THEN RETURN none            # outside: early out
  pick the axis with the SMALLEST depth
  normal := that axis, signed toward the point
  RETURN (that depth, normal)
```

**Invariants** — the shallowest axis wins: the nearest way out of a box is through the face it
is closest to. Choosing any other face pushes the point across the box.

The strict form is a predicate with no output and the full form returns the depth and normal;
both exist because the collider sometimes only needs to know, and sometimes needs to act.

## `FragmentonBoxTest` — segment against box

**Contract** — does a segment pass through an oriented box, and if so with what depth, normal
and contact point? A four-axis separating-axis test.

```text
FUNCTION segment_vs_box(p1, p2, box) -> optional<(depth, normal, position)>
  direction := normalize(p2 - p1)

  # 1. along the segment itself
  IF both endpoints are beyond the same end of the box's projection
    RETURN none

  # 2..4. the cross of the segment with each of the box's three axes
  FOR EACH box axis a
    axis := normalize(cross(direction, a))
    reach := the box's projection onto axis
    distance := signed distance from the box centre to the segment's line
    IF |distance| > reach THEN RETURN none
    depth[a] := reach - |distance|

  pick the axis with the SMALLEST depth
  normal   := that axis, scaled by its signed distance
  position := the closest approach of the segment's line to the corresponding
              box axis, offset by the normal
  RETURN (that depth, normal, position)
```

**Invariants** — the first test is the segment's own direction and it is a *containment* test
on an interval, not a distance test: it rejects only when both endpoints fall on the same side
of the box's extent, which correctly accepts a segment that spans the box entirely.

The shallowest cross axis wins, for the same reason as above.

**Notes** — the normal is deliberately left *unnormalised and scaled by the signed distance*,
so it carries both direction and sign in one value; the caller adds it to the contact position
to move the point onto the box's surface. That is compact and load-bearing rather than sloppy:
the caller needs exactly that offset.

The closest-approach helper (`CrossProjLine` and its two variants) exists in three forms
differing only in whether each argument is a free vector or a column of a rotation matrix. In a
rebuild there is one function; the three exist because the matrices are stored column-strided
and the arithmetic differs.

The bounded variant additionally rejects when the closest approach falls outside the segment,
or beyond a given half-extent along the box axis, and reports failure rather than a point.
That is what the box-versus-triangle edge case needs: a crossing that is off the end of either
edge is not a contact.

**Notes** — the file also carries a large commented-out earlier attempt at the segment test
that tried to fold the first axis into the same loop as the other three. It was abandoned and
nothing depends on it.
