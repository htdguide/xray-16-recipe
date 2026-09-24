# src/xrGame/ik_collide_data.h

> The three points on a foot that are tested against the ground, and the result of testing them.

**Needs** — [`xrCore/_fbox.h`](../xrCore/_fbox.h.md)
**Used by** — [`IKFoot.cpp`](IKFoot.cpp.md) · [`IKLimb.cpp`](ik/IKLimb.cpp.md) · [`IKLimb.h`](ik/IKLimb.h.md) · [`ik_foot_collider.cpp`](ik_foot_collider.cpp.md) · [`ik_foot_collider.h`](ik_foot_collider.h.md)
**Tier floor** — T2: a geometry record

## Purpose

A foot is not a point. Placing it on uneven ground requires knowing which *part* of it hit
first — the toe, the heel, or the outer edge — because that determines whether the foot
should pivot, lift, or roll. This file defines the three-point abstraction of a foot and the
record a ground test produces. It has no implementation file; the records are the contract.

## State

```text
ENUM CollidePoint  toe | heel | side | none

RECORD FootGeometry
  toe  : vector     # positions in the foot bone's own space
  heel : vector
  side : vector

RECORD CollideResult
  point     : CollidePoint = toe   # which of the three made contact
  plane     : plane                # the surface it hit: normal and offset
  pick_dir  : vector = (0,-1,0)    # the direction the test was run along
  collided  : bool                 # whether anything was hit at all
```

**Invariants** — three points, and exactly three. Two would not distinguish a foot rolling
sideways off a kerb from one pitching forward off a step; four would require deciding which
pair is diagonal. The three chosen span the two axes a foot actually articulates about: toe
and heel span pitch, side spans roll.

The result carries a **plane**, not a point. The solver needs the surface orientation to
align the foot with it, and a contact point alone would require a second query to get it.

The default contact point is the toe rather than "none", so that a result which was never
filled in behaves as a toe contact — the most common case — instead of as an invalid one.

## `is_valid`

**Contract** — whether the three points have been set. Implemented as a containment test
against a box spanning half the representable range.

**Notes** — the three points are initialized to the most negative representable value in
every component, and "valid" means "inside a very large box". That is a sentinel encoded in
the value rather than a separate flag, and it exists because the geometry is embedded by
value inside a per-limb record that is created before the skeleton is known. A rebuild with
an optional type should use one; what must survive is that an unset geometry is detectable,
because using one produces a foot placed at infinity.

The box is half the representable range rather than the whole of it so that the containment
test itself cannot overflow.
