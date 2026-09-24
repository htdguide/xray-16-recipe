# src/xrGame/ActorMountedWeapon.cpp

> The one transition by which the actor enters and leaves anything it can sit in or operate — a vehicle, a mounted gun, a turret.

**Needs** — [`Actor.h`](Actor.h.md) · [`holder_custom.h`](holder_custom.h.md) · [`Car.h`](Car.h.md) · [`ActorEffector.h`](ActorEffector.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`xrEngine/CameraBase.h`](../xrEngine/CameraBase.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a state transition that creates and destroys a physics body

## Purpose

*Holder* is the engine's word for any object the actor can occupy: a car's seat, a mounted
machine gun, a turret. While the actor is in a holder, the holder — not the actor's own
movement code — decides where the actor is, and the actor's walking collision capsule must
not exist, because two physics representations of one body fight each other. This file
owns that swap in both directions, and it is one function precisely so that the two
directions cannot drift apart.

The file's name is historical: it handles every holder, not only mounted weapons.

## State

`Stateless.` The actor's `holder` reference is the entire state and lives on the actor.

## `use_HolderEx`

**Contract** — toggles occupancy. Called with the holder the actor is looking at, or with
nothing when the actor asked to get out of whatever it is in. Returns whether the request
was consumed — a false return means the actor was not in a holder and the candidate could
not be entered, so the *use* action should fall through to whatever else it might mean.

**Invariants** — the actor's walking capsule exists if and only if the actor is not in a
holder. Entering destroys it; leaving recreates it. Both scripts and the interface observe
the transition through a callback fired with the holder's script facade.

```text
FUNCTION use_holder(candidate, forced) -> bool
  IF already in a holder THEN
    IF the holder is a car THEN
      detach_from_vehicle()                  # cars have their own exit path: animation, door, dismount point
      RETURN true

    IF holder.exit_is_locked THEN RETURN true   # consumed, but refused — a locked turret keeps the actor

    # Leaving is allowed only with no candidate at all, or when the candidate
    # IS the holder the actor is in. Pointing at a second holder from inside
    # the first does nothing, rather than teleporting between them.
    IF candidate is none OR candidate is the current holder THEN
      holder.detach_actor()
      fire script callback: detached from vehicle, with holder's facade
      recreate the actor's walking capsule
      holder = none
    RETURN true

  IF candidate exists AND NOT candidate.entry_is_locked THEN
    IF candidate.use(from: camera position, along: camera direction,
                     against: actor's centre) THEN
      IF candidate.attach_actor(this) THEN
        destroy the actor's walking capsule      # before anything else touches position
        holder = candidate
        drop the walk-bob camera effector        # the holder drives the camera now
        fire script callback: attached to vehicle, with holder's facade
        RETURN true

  RETURN false
```

**Notes**

- Entry is a two-stage handshake and both stages can refuse. The first asks the holder
  whether the *ray the player is looking along* actually hits a usable part of it — the
  driver's door rather than the bonnet — and the second asks whether the holder will take
  this occupant at all. Splitting them lets a holder accept a look from one angle only.
- The camera effector for walking bob is removed on entry and never restored on exit here.
  It is re-created by the actor's own movement code on the first frame it walks again.
- There is an unreachable branch: after the in-a-holder case has returned, the code asks
  again whether the (now known null) holder is a car and attaches a vehicle. It cannot
  run. A rebuild should omit it; it is a remnant of an earlier structure where vehicle
  entry was distinguished from holder entry at the top.
- The *forced* parameter is accepted and never consulted. A caller that wants to eject an
  occupant past a locked exit cannot do so through this path.
