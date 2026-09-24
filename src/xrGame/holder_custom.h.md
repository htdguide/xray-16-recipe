# src/xrGame/holder_custom.h

> The interface anything a player can climb into must satisfy: a vehicle, a mounted gun, a fixed camera — anything that takes over the actor's input, camera and movement.

**Needs** — [`Actor.h`](Actor.h.md) · [`Inventory.h`](Inventory.h.md) · [`CameraBase.h`](../xrEngine/CameraBase.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`ActorInput.cpp`](ActorInput.cpp.md) · [`ActorMountedWeapon.cpp`](ActorMountedWeapon.cpp.md) · [`ActorVehicle.cpp`](ActorVehicle.cpp.md) · [`Actor_Events.cpp`](Actor_Events.cpp.md) · [`Actor_Network.cpp`](Actor_Network.cpp.md) · [`AnselManager.cpp`](AnselManager.cpp.md) · [`Car.cpp`](Car.cpp.md) · [`Car.h`](Car.h.md) · [`HolderEntityObject.cpp`](HolderEntityObject.cpp.md) · [`HolderEntityObject.h`](HolderEntityObject.h.md) · [`WeaponStatMgun.cpp`](WeaponStatMgun.cpp.md) · [`WeaponStatMgun.h`](WeaponStatMgun.h.md) · [`holder_custom.cpp`](holder_custom.cpp.md) · [`holder_custom_script.cpp`](holder_custom_script.cpp.md) · _and 2 more_
**Tier floor** — T2: an interface plus two fields of state

## Purpose

When the player gets into a car or onto a mounted machine gun, the actor stops steering
itself: input, the camera, and the question of whether a weapon may be held all transfer to
the thing being ridden. This interface is that transfer contract. It is a mix-in rather than
a base class because the things that implement it — a car, a helicopter turret, a mounted
weapon — are already entities of unrelated kinds.

Since this is an abstract interface, what it *demands of an implementor* is the substance,
and it lives here rather than in [`holder_custom.cpp`](holder_custom.cpp.md), which holds
only the two-field attachment bookkeeping.

## State

```text
RECORD Holder
  owner       : optional<GameObject>   # who is riding; absent means empty
  owner_actor : optional<Actor>        # the same, when the rider is the player
  enter_locked : bool                  # scripts may forbid boarding
  exit_locked  : bool                  # ... and disembarking
  use_action   : text                  # the localized prompt shown when aiming at it
```

**Invariants** — the two owner references are set and cleared together and are two views of
one thing. The separate actor reference exists because almost every implementor needs the
rider *as the player* — to drive its camera, read its input, suspend its weapon — and only a
few need it as a generic object. A rebuild with a richer type system will collapse them.

Occupancy is defined as "the owner reference is set". There is no separate occupied flag, so
attachment and occupancy cannot disagree.

## What an implementor must provide

### Input

**Contract** — the holder receives raw input while occupied: mouse motion, keyboard press,
release and hold, controller press, release and hold with the axis state, and controller
attitude change. All are demanded, none has a default.

**Invariants** — the holder gets input *instead of* the actor, not in addition to it. An
implementor that wants the actor's normal bindings to keep working must forward them itself.
Press, release and hold are distinguished because vehicle controls are analogue in effect: a
steering key held is not a steering key tapped.

### Camera

**Contract** — `Camera` returns the camera the view is driven from while occupied, and
`cam_Update` advances it with the frame's elapsed time and a field of view. Both are
demanded.

**Invariants** — the holder owns the camera entirely. The actor's own camera stack is not
consulted while a holder is occupied, which is why a vehicle can have a camera that collides
with its own bodywork without the actor's collision logic interfering.

### Presentation policy

**Contract** — three questions the actor asks every frame and the holder answers:

- `allowWeapon` — may the rider hold a weapon while riding. A car driver may not; a
  passenger may.
- `HUDView` — is the first-person item view drawn. A mounted gun draws its own model and
  answers no.
- `UpdateEx` — the per-frame call the *rider* makes into the holder, carrying the field of
  view. Has an empty default, so a holder that needs nothing per frame says nothing.

### Boarding and leaving

**Contract** — `Use` is the interaction: given the user's position, aim direction and foot
position, the holder decides whether this is a valid boarding attempt and answers. The three
positions are all needed because a vehicle is entered at a *door*: aim decides which seat,
the eye position decides reach, and the foot position decides which side of the vehicle the
user is standing on.

`ExitPosition` is where the rider is placed on leaving, and `ExitVelocity` the velocity they
are given — zero by default, non-zero for a moving vehicle, so that stepping out of a
speeding car does not teleport a stationary body into the road.

The enter and exit locks are plain settable flags, read by the interaction code. They exist
so a mission script can seal a vehicle in either direction.

### Inventory

**Contract** — `GetInventory` returns the holder's own inventory, which is what makes a
vehicle a container as well as a ride.

### Scripted control

**Contract** — `Action` and two `SetParam` entry points, all with empty defaults: a generic
command channel by numeric identifier so that scripts can drive implementor-specific
behaviour (a turret's target, a car's engine) without the interface knowing what any of it
means.

**Notes** — the identifiers are implementor-defined and appear as bare numbers in scripts.
That is an untyped seam a rebuild can improve on, but the numbers are in shipped scripts and
are therefore frozen per implementor.
