# src/xrPhysics/IActivationShape.h

> The three ways the game asks "find me a spot for this volume that is not inside
> a wall".

**Needs** — [`IActivationShape.cpp`](IActivationShape.cpp.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md)
**Used by** — [`Explosive.cpp`](../xrGame/Explosive.cpp.md) · [`PhysicsShellHolder.cpp`](../xrGame/PhysicsShellHolder.cpp.md) · [`IActivationShape.cpp`](IActivationShape.cpp.md)
**Tier floor** — T2: three procedure declarations.

## Purpose

Declares the surface implemented in [`IActivationShape.cpp`](IActivationShape.cpp.md). It
exists so that the game layer can run an *activation* — see
[`PHActivationShape.cpp`](PHActivationShape.cpp.md) for what that is — without naming the
physics types involved.

## exported units

- **`ActivateShapeExplosive`** — settle a box at an explosion's origin and report both where
  it ended up and how large it could grow. Used to place a blast volume that is not inside
  geometry.
- **`ActivateShapePhysShellHolder`** — settle a box for an object that is about to become
  physical, inheriting that object's collision group so it does not fight its own parts.
- **`ActivateShapeCharacterPhysicsSupport`** — settle a box for a creature that is about to
  gain (or change) a physical body, optionally ignoring other characters and optionally
  taking the object's orientation. This is the one that returns whether it *succeeded*.
