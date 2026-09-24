# src/xrGame/trajectories.h

> Declares the ballistic line-of-flight test and the record its diagnostic overlay draws.

**Needs** — [`trajectories.cpp`](trajectories.cpp.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`CustomMonster.h`](CustomMonster.h.md) · [`control_jump.cpp`](ai/monsters/control_jump.cpp.md) · [`ai_stalker_fire.cpp`](ai/stalker/ai_stalker_fire.cpp.md) · [`trajectories.cpp`](trajectories.cpp.md)
**Tier floor** — T2: a geometry query signature.

## Purpose

Declares the surface implemented in [`trajectories.cpp`](trajectories.cpp.md). The shape of
the one exported operation is itself the decision worth recording, because six of its
arguments are there for reasons a rebuilder would otherwise have to rediscover.

## State

```text
RECORD TrajectoryPick        # one segment of the walk, for the diagnostic overlay only
  center                 : vector
  x_axis, y_axis, z_axis : vector
  sizes                  : vector
  invert_x, invert_y, invert_z : bool
```

The three inversion flags are declared and never written by anything in this module.

## `trajectory_intersects_geometry`

**Contract** — is the arc from a start to an end, launched at a given velocity and lasting a
given flight time, obstructed by level geometry? True means obstructed.

Beyond the arc itself it takes:

- **the thrower**, which is excluded from the query — an object always intersects its own
  starting point;
- **one further object to ignore**, so that a creature can throw past the squadmate or the
  vehicle it is standing on;
- **scratch space for the collision results**, supplied by the caller so that a test run per
  creature per decision allocates nothing;
- **an output collision position**, filled only in ray mode;
- **two optional diagnostic outputs** — the boxes walked, and the triangles hit — which are
  what the overlay draws;
- **a box size**, zero for a ray and non-zero to sweep a volume. A grenade and a jumping
  creature both have width; a test of the centre line alone would have them clip doorframes.

**Invariants** — the caller must supply the flight time and the velocity consistently with
the end position, because the routine derives neither from the other: it walks the arc the
velocity and time describe and compares where it lands against the end it was given.
