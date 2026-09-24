# src/xrPhysics/IPhysicsShellHolder.h

> Everything the physics module is allowed to know about the game object that owns
> a body — the inversion that keeps physics from depending on the game.

**Needs** — [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHCollisionDamageReceiver.cpp`](../xrGame/PHCollisionDamageReceiver.cpp.md) · [`PhysicsShellHolder.cpp`](../xrGame/PhysicsShellHolder.cpp.md) · [`PhysicsShellHolder.h`](../xrGame/PhysicsShellHolder.h.md) · [`ActorCameraCollision.cpp`](ActorCameraCollision.cpp.md) · [`ActorCameraCollision.h`](ActorCameraCollision.h.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`IActivationShape.cpp`](IActivationShape.cpp.md) · [`IActivationShape.h`](IActivationShape.h.md) · [`IClimableObject.h`](IClimableObject.h.md) · [`IColisiondamageInfo.h`](IColisiondamageInfo.h.md) · [`IElevatorState.h`](IElevatorState.h.md) · [`IPHCapture.h`](IPHCapture.h.md) · [`MovementBoxDynamicActivate.cpp`](MovementBoxDynamicActivate.cpp.md) · [`PHActivationShape.h`](PHActivationShape.h.md) · _and 17 more_
**Tier floor** — T3: a list of questions one subsystem asks another. Nothing about it needs
a manual tier.

## Purpose

`xrPhysics` is built before `xrGame` and must never name a game class, yet almost every
physical decision needs to ask something about the object it is simulating: where is it,
what is it called, is it the player, may a bullet hit it, who do I tell when it takes
damage. This header is the whole of that question set, written as an abstract surface the
game layer implements on its own object type. It is the concrete form of the
`xrEngine ⇄ xrPhysics` cycle break named in the
[build order](../../SYSTEM-REQUIREMENTS.md#7-build-order).

Read as a list, it is also a census of every place physics reaches upward — which makes it
the most useful page in the chapter for a rebuilder deciding where to draw the module line.

## `IPhysicsShellHolder`

**Contract** — implemented by any game object that can own a physics shell. The demands fall
into five groups.

**Identity and transform.** The object's world transform and position by mutable reference
(physics *writes* these — see the synchronization contract below), its name, its visual
name, its configuration section name, and its numeric id. The names exist only for
diagnostics; the id is the network and save-game handle.

**Owned sub-objects.** The object's skeleton (needed to map bones to bodies), its collision
form (the engine-side collision proxy, which physics must keep in step with the body), its
physics shell by *mutable reference* (so physics can null it out when it tears the shell
down), its active capture ([`IPHCapture.h`](IPHCapture.h.md)), its sound player, and its
collision-damage receiver.

**Classification predicates.** Is this an actor, a stalker, an inventory item; does it
collide with bullets; does it collide with the actor's camera; does it have a parent object.
These are asked from inside contact callbacks at the rate of thousands per step, which is
why they are predicates on the holder rather than type tests — the physics side must not
know the class hierarchy that answers them.

**Commands physics issues upward.** Deactivate/activate the object's processing, tell it its
spatial position changed, enable change notification, tell it its physics just went to
sleep, hide all its weapons (the capture path does this), enable or disable its movement
collision (the camera-collision path does this while it probes), and a hook that lets the
object scale a bounce damage factor.

**The damage path.** A callback interface that turns a collision into a hit: given the
object, a minimum and maximum contact speed, and mutable contact-speed and hit-level
outputs, it decides what damage the impact does. Physics computes the *kinematics* of the
impact; the game decides what that means.

**Invariants** — the transform reference handed out must remain valid for as long as the
shell exists; physics writes through it during the write-back phase of a step, not at
arbitrary times.

## `ICollisionHitCallback`

**Contract** — one call: given the struck object, the allowed contact-speed window, and
mutable contact-speed and hit-level values plus the collision damage description, produce
the hit. Installed per object; the physics side only ever invokes it.

## Notes

The synchronization contract this header encodes is worth stating plainly, because it is
scattered across the implementation and written down nowhere else:

- Between steps, the **game** owns an object's transform. It may teleport the object, and
  physics is told through the shell's transform-setting entry points.
- During a step, the **solver** owns the transform of every enabled body. The game must not
  write it; the `Processing` flag on [`IPHWorld.h`](IPHWorld.h.md) exists to catch attempts.
- After the last step of a frame, physics writes interpolated transforms back into the
  holder's matrix and bone array, then calls `ObjectSpatialMove` so the engine's spatial
  index and collision proxy follow. A disabled (sleeping) body writes nothing, which is why
  the game may move a sleeping object freely.

The debug-only `dump` entry, which renders an object's physical state as text at several
levels of detail, is a diagnostic and not part of the contract; it exists because
reproducing a physics bug from a log is otherwise impossible.
