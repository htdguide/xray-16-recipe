# src/xrPhysics/Physics.h

> Declares the module's shared free functions: contact generation, force and velocity clamping, collision energy, and the world boundary test.

**Needs** — [`Physics.cpp`](Physics.cpp.md) · [`PhysicsShell.h`](PhysicsShell.h.md) · [`PHObject.h`](PHObject.h.md) · [`PHWorld.h`](PHWorld.h.md) · [`PHInterpolation.h`](PHInterpolation.h.md) · [`PHContactBodyEffector.h`](PHContactBodyEffector.h.md) · [`BlockAllocator.h`](BlockAllocator.h.md) · [`PhysicsCommon.h`](PhysicsCommon.h.md) · [`phvalide.h`](phvalide.h.md) · [`dcylinder/dCylinder.h`](dcylinder/dCylinder.h.md)
**Used by** — [`IActivationShape.cpp`](IActivationShape.cpp.md) · [`PHAICharacter.cpp`](PHAICharacter.cpp.md) · [`PHCharacter.cpp`](PHCharacter.cpp.md) · [`PHElement.cpp`](PHElement.cpp.md) · [`PHFracture.cpp`](PHFracture.cpp.md) · [`PHIsland.cpp`](PHIsland.cpp.md) · [`PHJoint.cpp`](PHJoint.cpp.md) · [`PHObject.cpp`](PHObject.cpp.md) · [`PHShell.cpp`](PHShell.cpp.md) · [`PHShellSplitter.cpp`](PHShellSplitter.cpp.md) · [`PHSimpleCharacter.h`](PHSimpleCharacter.h.md) · [`PHSplitedShell.cpp`](PHSplitedShell.cpp.md) · [`PHWorld.h`](PHWorld.h.md) · [`PHWorldScript.cpp`](PHWorldScript.cpp.md) · _and 4 more_
**Tier floor** — T1: the declared surface reaches into the dynamics library's body and constraint types.

## Purpose

The module's shared toolbox header. Everything with substance is in
[`Physics.cpp`](Physics.cpp.md); this file declares it and carries three small decisions of its own.

## Exported units

- `NearCallback(a, b, shape_a, shape_b)` — generate the contacts between two physics objects and
  merge their islands if any were produced.
- `CollideStatic(shape, object)` — the same against the level's triangle soup.
- `BodyCutForce(body, linear_limit, angular_limit)` — clamp an already-applied force and torque so
  the resulting acceleration cannot exceed the limits.
- `body_angular_accel_from_torque(body, torque)` — invert the inertia tensor to get angular
  acceleration.
- `FixBody(body)` — make a body immovable by giving it an absurd mass and inertia and zeroing its
  state.
- `mass_sub(a, b)` — subtract one mass distribution from another, in place. The inverse of the
  library's mass-add; needed by breakables, which must split a mass into two parts.
- `E_NLD(body, body, normal)` — the kinetic energy lost in a perfectly inelastic collision between
  two bodies along a normal.
- `apply_gravity_accel(body, accel)` — apply an acceleration as a mass-scaled force.
- `contact_position(constraint)` — the world-space point of a contact constraint.
- `ContactGroup`, `ContactFeedBacks`, `ContactEffectors` — the three per-step pools: the
  constraints themselves, their force-feedback records, and the per-body accumulated contact
  effects. All three are emptied at the end of each step.
- `phBoundaries` and `PhOutOfBoundaries(point)` — the world box, and the escape test.

## Notes

`PhOutOfBoundaries` tests **only the lower bound**. An object that flies up or sideways out of the
level is not considered escaped; one that falls below the floor is. That asymmetry is correct for
the failure it guards: physics blow-ups eject downward through geometry far more often than
upward, and an object thrown high legitimately comes back.

The two constants `fix_ext_param` (10 000) and `fix_mass_param` (100 000 000) are the extent and
mass given to a "fixed" body. They are not a physical description — they are large enough that any
impulse a game object can deliver produces no observable motion, and small enough to stay far from
the float range's edge. A rebuild with a real static-body concept should use that instead; this
trick exists because the dynamics library's static bodies cannot be un-fixed later, and the engine
needs to release a fixed body back into motion (`release_fixed` in
[`PHElement.cpp`](PHElement.cpp.md)).
