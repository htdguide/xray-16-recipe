# src/xrPhysics/SpaceUtils.h

> Turns a collision space's axis-aligned bounds into the centre, half-extent and radius the engine's spatial index wants.

**Needs** — [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [`utils/xrMiscMath`](../utils/xrMiscMath/README.md)
**Used by** — [`PHActivationShape.cpp`](PHActivationShape.cpp.md) · [`PHShell.cpp`](PHShell.cpp.md) · [`PHSimpleCharacter.cpp`](PHSimpleCharacter.cpp.md) · [`PHSplitedShell.cpp`](PHSplitedShell.cpp.md) · [`PHStaticGeomShell.cpp`](PHStaticGeomShell.cpp.md)
**Tier floor** — T1: reads the dynamics library's internal bounds array rather than asking through its public surface.

## Purpose

A physics shell owns a collision space containing all of its geometries. The engine's spatial index
— the structure that decides which objects are near which — wants a centre, a half-extent vector and
a radius. This converts one to the other.

It exists as its own file because it is the one place that reaches *inside* the dynamics library
rather than through it: the library's public surface exposes bounds per geometry but not per space,
so the code asks the space to recompute its bounds and then reads the array directly. In a rebuild
that is either a one-line call on a well-designed dynamics seam, or a loop over the shell's
geometries — both are better than what is here, and the file vanishes.

## `spatial_pars_from_space`

**Contract** — forces the space to recompute its axis-aligned bounds, then derives the centre as the
midpoint of each axis, the half-extent as the distance from centre to the positive face, and the
radius as the **largest single half-extent**. Recomputing the bounds is not free — it walks every
geometry in the space — so this is called when an object's spatial registration is refreshed, not
per step.

```text
FUNCTION spatial_pars_from_space(space) -> (center, half_extent, radius)
  bounds = space.recompute_bounds()          # [lo.x hi.x lo.y hi.y lo.z hi.z]
  center      = midpoint of each axis
  half_extent = hi - center, per axis
  radius      = max(half_extent.x, half_extent.y, half_extent.z)
```

**Notes** — the radius is the largest half-extent, not the diagonal, so the sphere it describes does
**not** contain the box: a corner of the bounds sticks out by up to a factor of the square root of
three. That is a deliberate under-estimate, not an oversight — the value feeds a broad-phase
bucketing decision where a slightly tight sphere costs an occasional missed neighbour at the
extreme corner and a too-loose one costs work on every query, and the engine chose the cheap side.
A rebuild that uses this radius for anything that must be *conservative* — a culling test, a
trigger volume — has to take the diagonal instead.
