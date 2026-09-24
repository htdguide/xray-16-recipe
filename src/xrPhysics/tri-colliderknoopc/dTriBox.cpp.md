# src/xrPhysics/tri-colliderknoopc/dTriBox.cpp

> Box against triangle: a thirteen-axis separating-axis test, and the three-point manifold that
> lets a box lie flat on a surface without rocking.

**Needs** — [`dTriBox.h`](dTriBox.h.md) · [`dcTriListCollider.h`](dcTriListCollider.h.md) · [`dTriColliderCommon.h`](dTriColliderCommon.h.md) · [`../ExtendedGeom.h`](../ExtendedGeom.h.md) · [Seam: Rigid-body dynamics](../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`dTriBox.h`](dTriBox.h.md)
**Tier floor** — T1: separating-axis arithmetic over raw coordinate arrays, writing directly
into the solver's contact array.

## Purpose

Boxes are what most of the world's props are made of — crates, doors, furniture, debris — and
this is how they rest on the floor. Two routines: the ordinary case, where the box is in front
of a triangle, and the recovery case, where it is behind one.

The ordinary case is a full separating-axis test. The recovery case is not, and the difference
between them is the same split as everywhere else in this directory: *outside the world, be
careful; inside the world, push out and do not think*.

## Stateless.

## `dTriBox` — the ordinary case

**Contract** — report up to three contacts between an oriented box and a triangle the box's
centre is in front of.

The axis set is thirteen:

```text
  code 0       the triangle's plane normal
  codes 1..9   the box's three face normals, each tested against the three
               triangle vertices separately
  codes 10..18 the cross of each triangle edge with each box axis
```

**Invariants** — three rules govern the search and all three are load-bearing.

*A box face axis only counts if all three triangle vertices are on the same side of the box's
centre plane along it.* A triangle straddling the box along that axis is not separated by it
and its "depth" is meaningless.

*If any candidate axis separates the pair, the answer is immediately "no overlap".* This is
what makes a thirteen-axis test cheap in the common case.

*A cross-product axis must beat the current best by five percent to displace it.* Plain
"smaller depth wins" makes the chosen normal flicker between a face and an edge from step to
step as the box settles, and the box buzzes. The five percent is hysteresis, and the number is
a tuning choice with no derivation — it is small enough not to pick a visibly wrong normal and
large enough to stop the oscillation.

Additionally a cross axis is only adopted if the closest approach of the triangle edge to the
box edge actually lies on both of them (the bounded helper in
[`dTriBox.h`](dTriBox.h.md)), and degenerate cross products — the edge is parallel to the box
axis — are skipped rather than normalised.

**Contact synthesis** by winning code:

- **code 0, the triangle's face** — the box is lying on the surface. One contact at the box's
  deepest vertex, then up to two more found by walking from that vertex along the box's two
  *least*-projected edges, each accepted only while its running depth stays positive; and then
  one more from the second of those, giving a diagonal. This is the manifold that stops a
  crate rocking on a floor: three points spanning a face rather than one point under a corner.
- **codes 1–9, a box face** — one contact at the triangle vertex that won, with the box face's
  normal.
- **codes 10–18, an edge cross** — one contact at the closest-approach point already computed
  while testing the axis.

**A final gate**: if the chosen normal points along the triangle's own normal rather than
against it, the whole result is discarded. A contact that would push the box *into* the
surface is worse than no contact.

Every emitted contact names the mesh as the first shape and the box as the second, carries the
triangle's material in its surface parameters, and the box's contact callback is fired once
with the triangle.

**Notes** — the "walk along the least-projected edges" construction is inherited from the
dynamics library's own box-versus-plane routine and is marked in the original as not quite
right for a triangle: it does not check that the extra points are actually over the triangle,
only that they are still penetrating the plane. In practice the neighbouring triangle produces
the missing support, so the error is invisible. A rebuild may either add the containment check
or keep the note.

## `dSortedTriBox` — the recovery case

**Contract** — the box is *behind* the triangle's plane. No axis search: the normal is the
triangle's plane normal, negated.

```text
FUNCTION box_vs_plane(tri_normal, triangle, distance, box, mesh, max_contacts) -> int
  reach := the box's projection onto tri_normal
  depth := reach - distance
  IF depth < 0 THEN RETURN 0

  first contact := the box's deepest vertex along the normal
  then, as above, walk along the box's two least-projected edges for a second and
  a third contact, each accepted only while its running depth is positive,
  stopping at the caller's maximum (never more than three)
  every contact carries tri_normal and the triangle's material
```

**Notes** — the same manifold construction as the ordinary case, without the containment
question. A box being extracted from a wall needs the three-point manifold just as much as one
resting on a floor, because a single-point push rotates it while it comes out.

The contact count is capped at three regardless of what the caller asks for. Three is the
minimum that constrains a flat face and, past it, the solver gains nothing from a redundant
fourth: it has to reconcile the extras and the box settles no faster.
