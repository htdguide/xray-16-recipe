# src/xrPhysics/ph_valid_ode.h

> Tests whether a body's state is still made of finite numbers.

**Needs** — [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [`MathUtils.h`](MathUtils.h.md)
**Used by** — [`MovementBoxDynamicActivate.cpp`](MovementBoxDynamicActivate.cpp.md) · [`PHIsland.cpp`](PHIsland.cpp.md) · [`PHShellSplitter.cpp`](PHShellSplitter.cpp.md) · [`PHSimpleCharacter.cpp`](PHSimpleCharacter.cpp.md) · [`PHValideValues.h`](PHValideValues.h.md)
**Tier floor** — T1: reads the dynamics library's state arrays field by field, including the padding layout of its rotation matrix.

## Purpose

A constraint solver that is fed a degenerate configuration — a zero-length joint axis, a zero mass,
two geometries at exactly the same point — can produce infinities and not-a-numbers, and once one
appears it propagates through every body sharing an island within a step or two. These predicates
are the tripwires: the world checks body state at the boundaries of each step and, when one fails,
takes the containment path in [`PHIsland.cpp`](PHIsland.cpp.md) rather than letting the poison
spread.

They are separate from the general validity helpers in [`MathUtils.h`](MathUtils.h.md) because they
read the *library's* representations — its vector, matrix, quaternion and mass layouts — rather than
the engine's.

## State

`Stateless.`

## The predicates

**Contract** — each answers "is every number in this thing finite?" and has no other effect. They
are cheap enough to run per body per step in a debug build and are compiled out of shipping builds
at most call sites.

```text
FUNCTION vector_is_valid(v)      -> bool    # three components
FUNCTION vector4_is_valid(v)     -> bool    # four components; also used for quaternions
FUNCTION rotation_is_valid(m)    -> bool    # the nine real entries of a 3x3 rotation
FUNCTION mass_is_valid(m)        -> bool    # scalar mass, centre of mass, inertia tensor
FUNCTION body_state_is_valid(b)  -> bool
    = rotation_is_valid(b.rotation)     AND vector_is_valid(b.position)
      AND vector_is_valid(b.linear_velocity)  AND vector_is_valid(b.angular_velocity)
      AND vector_is_valid(b.torque)           AND vector_is_valid(b.force)
```

**Invariants** — the rotation check reads nine of twelve stored values, skipping one per row. That
is not a bug: the library stores a 3x3 rotation as three rows of four, with the fourth entry of each
row unused padding so that each row is a whole wide-float register. A rebuild that stores rotations
densely checks nine of nine; a rebuild that keeps the padding must skip it, because the padding is
never written and may hold anything.

The body-state check deliberately covers *accumulators* — force and torque — as well as state.
A non-finite accumulated force has not affected the body yet, and catching it before the step is the
difference between clearing one body's force and losing the object.

**Notes** — the mass check exists because an inertia tensor built from a degenerate collision shape
(a zero-radius sphere, a box with a zero side) is the most common source of poison in practice.
A rebuild is better served by rejecting such shapes at build time — see the model verification in
[`PhysicsShell.cpp`](PhysicsShell.cpp.md) — and keeping this as the last line rather than the first.
