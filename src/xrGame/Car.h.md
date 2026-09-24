# src/xrGame/Car.h

> Declares the vehicle and its four nested part records — wheel, door, exhaust, sound — implemented across [`Car.cpp`](Car.cpp.md) and its five siblings.

**Needs** — [`Entity.h`](Entity.h.md) · [`script_entity.h`](script_entity.h.md) · [`holder_custom.h`](holder_custom.h.md) · [`PHSkeleton.h`](PHSkeleton.h.md) · [`PHDestroyable.h`](PHDestroyable.h.md) · [`DamagableItem.h`](DamagableItem.h.md) · [`PHCollisionDamageReceiver.h`](PHCollisionDamageReceiver.h.md) · [`DelayedActionFuse.h`](DelayedActionFuse.h.md) · [`Explosive.h`](Explosive.h.md) · [`hit_immunity.h`](hit_immunity.h.md) · [`CarLights.h`](CarLights.h.md) · [`CarDamageParticles.h`](CarDamageParticles.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`xrPhysics/PHUpdateObject.h`](../xrPhysics/PHUpdateObject.h.md) · [`xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md)
**Used by** — [`ActorAnimation.cpp`](ActorAnimation.cpp.md) · [`ActorCameras.cpp`](ActorCameras.cpp.md) · [`ActorInput.cpp`](ActorInput.cpp.md) · [`ActorMountedWeapon.cpp`](ActorMountedWeapon.cpp.md) · [`ActorVehicle.cpp`](ActorVehicle.cpp.md) · [`Car.cpp`](Car.cpp.md) · [`CarCameras.cpp`](CarCameras.cpp.md) · [`CarDamageParticles.cpp`](CarDamageParticles.cpp.md) · [`CarDoors.cpp`](CarDoors.cpp.md) · [`CarExhaust.cpp`](CarExhaust.cpp.md) · [`CarInput.cpp`](CarInput.cpp.md) · [`CarLights.cpp`](CarLights.cpp.md) · [`CarScript.cpp`](CarScript.cpp.md) · [`CarSound.cpp`](CarSound.cpp.md) · _and 7 more_
**Tier floor** — T3: a declaration only

## Purpose

Declares the vehicle. Its value as a page is the **inheritance list**, which is the widest
in the game and is itself the design: a car is simultaneously

- a **damageable entity** with health, a team and a death,
- a **script entity** a Lua sequence can drive,
- a **physics-step participant** that is called inside the solver's own callbacks,
- a **holder** the actor can be inside,
- a **physics skeleton** whose bones are joints and elements,
- a **damageable item** with a multi-level damage ladder,
- a **destructible** that can be swapped for a wreck model,
- a **collision-damage receiver** that is hurt by running into things,
- a **hit-immunity carrier** with a per-damage-type scale table,
- an **explosive**, and
- a **delayed-action fuse** that puts a timer between damage and explosion.

A rebuild composes these rather than inheriting them, but must keep every one of them:
each contributes behaviour the shipped vehicles depend on. The order in which their
lifecycle hooks run is the substance of [`Car.cpp`](Car.cpp.md).

Substance is split across
[`Car.cpp`](Car.cpp.md) (engine, gearbox, controls, lifecycle),
[`CarDoors.cpp`](CarDoors.cpp.md) (the door state machine and its geometry),
[`CarWheels.cpp`](CarWheels.cpp.md) (the wheel joints),
[`CarInput.cpp`](CarInput.cpp.md) (input and the script-action bridge),
[`CarCameras.cpp`](CarCameras.cpp.md) (the three cameras),
[`CarExhaust.cpp`](CarExhaust.cpp.md) (exhaust emitters),
[`CarSound.cpp`](CarSound.cpp.md) (the engine sound state machine) and
[`CarScript.cpp`](CarScript.cpp.md) (the script surface).

Exported units:

- `CCar` — the vehicle.
- `CCar::SWheel` — one wheel: a bone, a joint, a radius, per-wheel collision tuning
  (spring, damping and friction factors applied to every contact the wheel makes), and its
  own health. Its axis operations set a target velocity and a maximum torque on either the
  drive axis or the steer axis — that pair is the *entire* interface between the engine
  model and the rigid-body world.
- `CCar::SWheelDrive`, `SWheelSteer`, `SWheelBreak` — the three *roles* a wheel can hold,
  each a thin record pointing at a wheel plus the numbers that role needs. One wheel may
  hold all three roles.
- `CCar::SDoor` — one door: a hinge joint, an open and a closed angle, a state among
  opening/closing/opened/closed/broken, a torque, and the plane geometry used to answer
  "can a person pass through this doorway".
- `CCar::SDoor::SDoorway` — a second, unused attempt at the same doorway geometry; its
  trace operation is empty.
- `CCar::SExhaust` — a bone, a particle emitter and the physics element whose velocity the
  emitter inherits.
- `CCar::SCarSound` — the engine sound state machine: off, stalling, stopping, starting,
  driving, with a configured delay between the starter sound and the engine loop.
- `ECarCamType` — the three cameras: first-person, chase, free.
- `eStateDrive`, `eStateSteer` — the drivetrain and steering state each reduce to three
  values regardless of how they were commanded.
