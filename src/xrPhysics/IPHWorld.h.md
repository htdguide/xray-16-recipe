# src/xrPhysics/IPHWorld.h

> The port through which the engine drives the physics world — gravity, the fixed
> step, freezing, and the deferred-call queue — without knowing what a body is.

**Needs** — [`PhysicsExternalCommon.h`](PhysicsExternalCommon.h.md) · [`PHCommander.h`](PHCommander.h.md) · [`xrCDB/xrCDB.h`](../xrCDB/xrCDB.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`Actor_Network.cpp`](../xrGame/Actor_Network.cpp.md) · [`Artefact.cpp`](../xrGame/Artefact.cpp.md) · [`BlackGraviArtifact.cpp`](../xrGame/BlackGraviArtifact.cpp.md) · [`Car.cpp`](../xrGame/Car.cpp.md) · [`CarDamageParticles.cpp`](../xrGame/CarDamageParticles.cpp.md) · [`CarExhaust.cpp`](../xrGame/CarExhaust.cpp.md) · [`CarLights.cpp`](../xrGame/CarLights.cpp.md) · [`CarSound.cpp`](../xrGame/CarSound.cpp.md) · [`Explosive.cpp`](../xrGame/Explosive.cpp.md) · [`GamePersistent.cpp`](../xrGame/GamePersistent.cpp.md) · [`Level.cpp`](../xrGame/Level.cpp.md) · [`Level_network_compressed_updates.cpp`](../xrGame/Level_network_compressed_updates.cpp.md) · [`Level_network_messages.cpp`](../xrGame/Level_network_messages.cpp.md) · [`Level_network_start_client.cpp`](../xrGame/Level_network_start_client.cpp.md) · _and 8 more_
**Tier floor** — T2: it is a pure interface over a simulation clock; nothing here touches
layout or a device. The T1 floor of the chapter comes from what implements it.

## Purpose

This is the cycle-breaker named in [§7 of the requirements](../../SYSTEM-REQUIREMENTS.md#7-build-order):
the frame loop owns the clock and needs to advance physics, but must not see the dynamics
library's types. Everything the loop and the game layer legitimately need from the physics
world is listed here, and exactly one instance exists, reached through a global accessor
that is created once per level and destroyed with it.

The interface is deliberately narrow in one direction and wide in another: it exposes the
*schedule* (how many steps, how long a step, freeze/unfreeze) in full, and exposes the
*contents* (bodies, joints, contacts) not at all. A caller that wants to touch a body goes
through a shell ([`PhysicsShell.h`](PhysicsShell.h.md)), never through here.

## `IPHWorld`

**Contract** — the abstract physics world. An implementor must satisfy:

- **Gravity** is one scalar (the downward acceleration magnitude), readable and settable at
  run time; weather and script both change it.
- **Step scheduling.** The world runs on a *fixed* timestep, not the frame's elapsed time.
  Given a frame's elapsed milliseconds, the world reports how many whole steps it owes
  (`CalcNumSteps`) and then runs them. The running total of steps ever taken is exposed as a
  64-bit counter and is used as a simulation-time stamp by everything that needs to say
  "this happened at step N" — impulse expiry, disabling counters, capture timeouts. It must
  be monotonic and must not be derived from wall-clock time, because
  [conformance criterion 8](../../SYSTEM-REQUIREMENTS.md#6-conformance) requires the same
  input sequence to produce the same trajectories.
- **Freeze / unfreeze** stops the world advancing while leaving every object registered.
  This is not a pause for the player: it is used by the activation procedures
  ([`PHActivationShape.cpp`](PHActivationShape.cpp.md),
  [`MovementBoxDynamicActivate.cpp`](MovementBoxDynamicActivate.cpp.md)) which need to run
  the solver many times *outside* the game's timeline to push a newly spawned body out of
  the walls, then restore the world as if nothing happened.
- **A processing flag** that is true while a step is in progress. Callers assert on it: a
  large class of bugs in this chapter is a game-side callback mutating the world from inside
  a contact callback, and the flag is how those are caught.
- **Two default contact callbacks** — one for rigid bodies, one for characters — installed
  by the game layer to leave a mark (a decal, a sound) where something struck static
  geometry. The world holds them so that any geometry created without its own callback still
  produces the mark.
- **A step-time callback** reporting the wall-clock bracket of each step, for the profiler
  and for the network layer's timing.
- **Deferred calls**: a condition/action pair queued to be evaluated once per step
  ([`PHCommander.h`](PHCommander.h.md)). This is how script-side and game-side work gets onto
  the physics timeline rather than the frame timeline.
- **Statistics**: three timers — collision detection, integration, and character
  movement-plus-collision — reset each frame and dumped into the debug overlay.

**Invariants** — exactly one world exists at a time; creating a second without destroying
the first is a programming error. Every object registered in the world must be unregistered
before the world is destroyed (this is the physics half of the
[destroyed-entity invariant](../../SYSTEM-REQUIREMENTS.md#6-conformance)).

## `physics_world` / `create_physics_world` / `destroy_physics_world`

**Contract** — the module's lifecycle entry points, exported across the module boundary.
Creation takes the level's static collision database and the level's object list, because
the world needs both: the first to answer the mesh collider's queries, the second to walk
objects when the level unloads. A flag selects whether the world runs on its own thread.
`physics_world` returns the one instance, or nothing when no level is loaded — callers in
the game layer check it on every use, which is the honest signal that the world's lifetime
is shorter than theirs.

**Notes** — `destroy_object_space` sits here too, which is a layering accident: the static
collision database's wrapper is created by the engine and destroyed through a physics entry
point only because the physics world is the last thing holding a pointer to it. A rebuild
should give the level loader both halves.

## `PHWorldStatistics`

**Contract** — three per-frame accumulating timers with a frame-start / frame-end bracket.
Purely observational; no simulation decision reads them.
