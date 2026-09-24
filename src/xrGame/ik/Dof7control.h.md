# src/xrGame/ik/Dof7control.h

> Declares the closed-form inverse of a spherical-revolute-spherical chain: given where
> the end must be and how it must be oriented, recover the two ball joints and the one
> hinge between them.

**Needs** — [`math3d.h`](math3d.h.md)
**Used by** — [`Dof7control.cpp`](Dof7control.cpp.md) · [`limb.cxx`](limb.cxx.md) · [`limb.h`](limb.h.md)
**Tier floor** — T2. A record of matrices and cached circle geometry; no allocation.

## Purpose

Declares the surface implemented in [`Dof7control.cpp`](Dof7control.cpp.md), which carries
the substance.

The chain it inverts is stated in the source as one equation, and it is the equation the
whole directory exists to solve:

```text
G = R2 · S · Ry · T · R1
```

`R1` is the root ball joint (the hip), `T` the fixed root-to-hinge transform, `Ry` the
hinge (the knee, one degree about the *y* axis), `S` the fixed hinge-to-tip transform, and
`R2` the tip ball joint (the ankle). `G` is the goal. Seven unknowns, six equations, one
free parameter.

## Exported units

Setup:

- Initialize with the two constant link transforms, a **projection axis** and a **positive
  direction axis**. The first fixes where the swivel angle is measured *from*; the second
  fixes which way round it counts. Both are arbitrary but must never change afterwards,
  because the swivel angle is carried between frames.
- Read or replace either link transform after the fact.
- The chain's total reach, as the sum of the two link lengths.

Posing a problem, one of which must be called before anything else:

- Set a full goal — position and orientation — returning the hinge angle and whether the
  goal was feasible.
- Set a position-only goal, given a constant transform that relates the tip to the point
  being aimed at.
- Set an *aiming* goal: point a named axis of the tip at a target, with the hinge angle
  given rather than solved. Used for arms; the shipped game enables no arm limbs.

The free parameter, in both directions:

- Swivel angle from hinge position, and hinge position from swivel angle. Also a direct
  recomputation of the swivel circle for a supplied end position, which exists so the
  engine can ask "what swivel angle did the animator pose" against a goal it has not set.

Solving, once a goal is posed:

- Both ball joints from a swivel angle, or from a hinge position.
- The root ball joint alone, for the position-only problem.
- The root ball joint for the aiming problem.

And the swivel-parameterized forms, which are what makes the joint-limit analysis
possible:

- The root rotation as `cos ψ · C + sin ψ · S + O`, three matrices.
- Both rotations in that form, six matrices.

Finally a switch: **project the goal into the reachable workspace, or do not.** On by
default and never turned off by this engine.
