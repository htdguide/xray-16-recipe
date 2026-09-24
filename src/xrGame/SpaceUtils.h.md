# src/xrGame/SpaceUtils.h

> Derives a spatial-index bound — centre, half-extents and radius — from a dynamics-library collision space.

**Needs** — [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it reads a foreign library's internal bounds array in place

## Purpose

The engine's spatial database indexes every object by a centre and a radius, so that
visibility, senses and broadphase queries can find things near a point. A physics object's
extent is not the engine's to compute — it belongs to the dynamics library, which already
maintains a bounding box over the whole collision space of a jointed assembly. This one
function converts that box into the engine's own description.

It is a header-only inline because it sits on the per-frame path of every physics object
that moves, and the whole body is four arithmetic lines.

## State

`Stateless.`

## `spatialParsFromDGeom`

**Contract** — takes a collision space belonging to the dynamics seam, forces it to
recompute its axis-aligned bounds, and returns the box's centre, its half-extents, and a
radius that encloses it. Does not allocate, does not block, and must be called on the
thread that owns the dynamics world, because forcing the recompute mutates the library's
state.

```text
FUNCTION spatialParsFromDGeom(space) -> (centre, half_extents, radius)
  space.recompute_bounds()
  bounds <- space.bounds          # six reals: min and max per axis, interleaved
  centre <- midpoint of bounds per axis
  half_extents <- bounds.max - centre, per axis
  radius <- largest of the three half-extents      # see note
  RETURN (centre, half_extents, radius)
```

**Notes** — the radius is the **largest half-extent**, not the diagonal. It therefore does
*not* enclose the box: the corners stick out by up to the square root of three times the
radius. This is deliberate and it is a gameplay tuning decision disguised as geometry — a
spatial radius that enclosed the box would make every long thin object (a pipe, a fence
section, a corpse) register as present far beyond where it visibly is, and the senses and
broadphase would both over-report. Under-covering the corners costs a rare missed contact
at a box corner; over-covering costs constant false positives. A rebuild should make the
same choice knowingly rather than reproduce a formula.

The bounds are read as a flat array of six numbers laid out as (min x, max x, min y, max
y, min z, max z), and in the dynamics library's own precision, which may be wider than the
engine's. The conversion happens at the arithmetic. A rebuild whose dynamics seam reports
bounds as a pair of vectors needs none of this.

This file reaches directly into the dynamics library's internal headers rather than through
its public interface, which is the reason the whole file is quarantined here and not in the
physics module. A rebuild should ask the seam for the bounds through its published surface
and delete the quarantine.
