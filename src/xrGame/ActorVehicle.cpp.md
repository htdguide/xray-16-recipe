# src/xrGame/ActorVehicle.cpp

> Seats the actor in a car and gets it out again: swap the walking capsule for the car's physics, rebind the skeleton's aim callbacks, and hide the weapon.

**Needs** — [`Actor.h`](Actor.h.md) · [`Car.h`](Car.h.md) · [`holder_custom.h`](holder_custom.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`PHMovementControl.h`](PHMovementControl.h.md) · [`ActorEffector.h`](ActorEffector.h.md) · [`actor_anim_defs.h`](actor_anim_defs.h.md) · [`Inventory.h`](Inventory.h.md) · [`game_object_space.h`](game_object_space.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a state transition across the physics and animation subsystems

## Purpose

Entering a vehicle is the most invasive state change the actor undergoes: it stops being a
physically simulated walking body and becomes a passenger of one, its skeleton's aim
callbacks change meaning, its weapon must disappear, and its identity must be recorded in
a form that survives a save. All of that is here, paired so that the exit undoes the entry
field for field.

This file handles cars specifically. The general holder case — turrets, mounted guns — is
in [`ActorMountedWeapon.cpp`](ActorMountedWeapon.cpp.md), which delegates here when the
holder turns out to be a car.

## State

Nothing is owned here. The transition writes four fields on the actor:

```text
  holder     : the occupied object, or none
  holder_id  : entity identifier of the holder, or the invalid identifier
               # invariant: set iff holder is set. This is the SAVED half —
               # a pointer cannot be written to a save, an identifier can.
  wishful movement state : zeroed on entry, so queued walking input is discarded
  model yaw / torso yaw  : on exit, both are reset to the car camera's facing
```

## `attach_Vehicle`

**Contract** — seats the actor. Refuses silently when there is no vehicle, when the actor
already occupies something, or when the candidate is not a car. The car may also refuse,
in which case the half-applied holder reference is undone before returning.

**Invariants** — the walking capsule is destroyed *after* the car has accepted the actor,
never before: a refused entry must leave the actor walking. The order of the remaining
steps matters only in that the animation must be playing before the head callback is
installed, since the callback modifies the pose that animation produces.

```text
FUNCTION attach_vehicle(vehicle)
  IF vehicle is none OR already in a holder THEN RETURN
  IF vehicle is not a car THEN RETURN

  holder = vehicle
  IF NOT vehicle.attach_actor(this) THEN holder = none; RETURN

  play the car's driver idle pose for the car's driver-animation type
  reset the skeleton's bone callbacks to their defaults
  install the seated-head aim callback on the head bone     # replaces the standing spine bend
  destroy the walking capsule
  wishful movement state = 0
  holder_id = vehicle.entity_id
  hide the weapon, tagged as hidden-because-in-a-car
  tell the step manager no leg motion is playing            # no footsteps while seated
  fire script callback: attached to vehicle
```

**Notes**

- Resetting *all* bone callbacks and then installing one is the cheap way to remove the
  four standing spine-bend callbacks described in
  [`ActorAnimation.cpp`](ActorAnimation.cpp.md). A rebuild with a named callback set can
  remove them individually.
- The weapon-hide reason is a *tagged* state rather than a boolean, because several things
  can hide the weapon at once — a car, a conversation, a script — and each must be able to
  un-hide only its own reason.

## `detach_Vehicle`

**Contract** — gets the actor out. Refuses silently when not in a car. The exit can fail:
if the walking capsule cannot be created at the exit point — something is standing there —
the actor stays in the car and nothing is changed.

**Invariants** — the car's physics shell is temporarily *split apart* around the attempt to
create the capsule and reassembled whether the attempt succeeds or fails, because the
capsule would otherwise be created colliding with the car it is climbing out of. The
reassembly appears on every path out of the function, which is the invariant a rebuild must
preserve; a language with scoped cleanup expresses it better.

```text
FUNCTION detach_vehicle()
  IF not in a holder THEN RETURN
  IF the holder is not a car THEN RETURN

  car.physics_shell.splitter_holder_deactivate()
  ok = create the walking capsule
  car.physics_shell.splitter_holder_activate()       # always, both branches
  IF NOT ok THEN RETURN                              # stay seated

  holder.detach_actor()
  fire script callback: detached from vehicle
  place the actor at the car's exit position, with the car's exit velocity
  model yaw = torso yaw = desired model yaw = -(car camera yaw)   # face where you were looking
  holder = none; holder_id = invalid
  restore the standing bone callbacks
  play the standing leg idle and torso idle
  un-hide the weapon's car-hide reason
```

**Notes** — inheriting the car's velocity on exit is deliberate: stepping out of a moving
car should throw the actor, not teleport it to a standstill. It is also how the actor can
be killed by leaving a car at speed.

## `use_Vehicle`

**Contract** — the player's *use* action, routed at a vehicle. Returns whether the action
was consumed. When seated, a use gesture aimed at nothing or at the same car exits it —
and only if the car's own hit test agrees the gesture was aimed at a usable part. When on
foot, a use gesture aimed at a car enters it, dropping the walk-bob camera effector first;
if the hit test *fails* the action is still consumed, but a distinct "used a vehicle
without entering" script callback fires instead, which is what lets scripts react to a
player rattling a locked door.

**Notes** — the hit test is performed from the *camera's* position and direction rather
than from the actor's, because the player aims with the view.

## `on_requested_spawn`

**Contract** — seats the actor in a car that has just been spawned on request. This is how
a script can create a vehicle and put the player in it in one step, without racing against
the spawn's completion.
