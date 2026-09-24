# src/xrGame/ik/limb.h

> Declares the limb as a *joint-space* problem: the layer that turns the geometric
> solver's two rotation matrices into seven joint angles, and that owns the choice of
> swivel angle.

**Needs** — [`aint.h`](aint.h.md) · [`Dof7control.h`](Dof7control.h.md) · [`eulersolver.h`](eulersolver.h.md)
**Used by** — [`IKLimb.cpp`](IKLimb.cpp.md) · [`IKLimb.h`](IKLimb.h.md) · [`limb.cxx`](limb.cxx.md)
**Tier floor** — T2. A record holding a geometric solver, seven arcs and four interval
sets.

## Purpose

Declares the surface implemented in [`limb.cxx`](limb.cxx.md), which carries the
substance. It is the boundary the engine actually talks to —
[`IKLimb.cpp`](IKLimb.cpp.md) holds one of these and knows nothing below it.

The seven degrees of freedom are indexed `0..6`: three for the root ball joint, one for
the hinge at index 3, three for the tip ball joint. Every array in the type follows that
layout, and the joint-limit array is public because the debug draw reaches into it.

## Exported units

Setup:

- Initialize with the two constant link transforms, the Euler convention for each ball
  joint, the projection and positive-direction axes, and seven low/high limit pairs.
- Replace either link transform afterwards.
- The chain's reach.

Posing a goal, which must precede any solve:

- Set a full position-and-orientation goal, with joint limits on or off. With limits on it
  also computes the four feasible swivel-angle sets — one per combination of the two
  joints' solution families.
- Set a position-only goal, likewise, with two sets instead of four.
- Set an aiming goal.

Solving, each returning success, and optionally the swivel angle used and the hinge
position it corresponds to:

- Solve, choosing the swivel angle itself — the midpoint of the largest feasible stretch
  when limits are on, and zero when they are off.
- Solve for a *given* swivel angle. This is the entry point the engine uses.
- Solve for a given hinge position, which converts and delegates.
- Solve the aiming problem for a given swivel angle.

Queries:

- Swivel angle from a hinge position, against the currently posed goal; and the same
  against an arbitrary goal position without posing it, which is how the engine reads the
  animator's knee.
- Are these seven angles all within limits.
- Forward kinematics: compose seven angles back into the tip's transform.
- Retrieve the feasible swivel-angle sets, and a variant that returns the per-joint sets
  before they are intersected.
- Dump the swivel-parameterized matrices and limits to two files, for offline analysis.
