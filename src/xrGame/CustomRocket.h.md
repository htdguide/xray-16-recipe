# src/xrGame/CustomRocket.h

> Declares the flying, glowing, smoking rocket a launcher fires, implemented in [`CustomRocket.cpp`](CustomRocket.cpp.md).

**Needs** — [`physic_item.h`](physic_item.h.md) · [`xrPhysics/PHUpdateObject.h`](../xrPhysics/PHUpdateObject.h.md)
**Used by** — [`CustomRocket.cpp`](CustomRocket.cpp.md) · [`ExplosiveRocket.cpp`](ExplosiveRocket.cpp.md) · [`ExplosiveRocket.h`](ExplosiveRocket.h.md) · [`PhysicsShellHolder.cpp`](PhysicsShellHolder.cpp.md) · [`RocketLauncher.cpp`](RocketLauncher.cpp.md) · [`RocketLauncher.h`](RocketLauncher.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the rocket's flight half. It is a physics item — an inventory item with a body —
that also participates in the **physics step**, which is what lets its motor apply impulses
at the solver's own cadence rather than the frame's. Substance in
[`CustomRocket.cpp`](CustomRocket.cpp.md); the explosion is added by
[`ExplosiveRocket.h`](ExplosiveRocket.h.md).

Exported units:

- `CCustomRocket` — the projectile.
- `SRoketContact` — the one recorded contact: did it happen, where, and the surface normal.
  Recorded inside the solver's callback and acted on later, because nothing may change the
  world from inside that callback.
- `ERocketState` — inactive (in an inventory), engine (motor burning), flying (coasting),
  collide (stopped). One-way.
- `SetLaunchParams` — the launcher hands over a transform, a velocity and an angular
  velocity, plus arming the forced-detonation deadline.
- `activate_physic_shell`, `create_physic_shell` — build the body: a narrowed box with a
  large nose sphere and a small tail sphere.
- `StartEngine`, `StopEngine`, `UpdateEngine`, `UpdateEnginePh` — the motor, timed per frame
  and applied per physics step.
- `StartFlying`, `StopFlying`, `StartLights`, `StopLights`, `UpdateLights`,
  `StartEngineParticles`, `StartFlyParticles`, `StopEngineParticles`, `StopFlyParticles`,
  `UpdateParticles` — the trail.
- `Contact`, `PlayContact`, `ObjectContactCallback` — the collision: recorded in the
  solver's callback (including the correction that recovers the true surface point from a
  body the solver has already pushed out), acted on in the frame.
- `PhDataUpdate`, `PhTune` — the two physics-step hooks. Only the second does anything: it
  runs the motor.
- `AlwaysTheCrow` — always true: a rocket demands an unconditional per-frame update and is
  never rate-reduced by distance.
- `Useful` — true only while inactive, which is the test for whether a pooled rocket can be
  handed out again.
- `OnH_B_Chield`, `OnH_A_Chield`, `OnH_B_Independent`, `OnH_A_Independent` — attachment;
  leaving the launcher is what starts the flight.
- `UsedAI_Locations` — a rocket holds no navigation position.

**Notes** — the launcher is declared a friend, which is how it sets the launch parameters and
the launched flag without those being public. In a rebuild the launch is one call taking a
launch record; the friendship is an artefact.
