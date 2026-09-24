# src/xrGame/PHDestroyable.h

> Declares the mixin that lets a physically simulated object break into separately spawned pieces, implemented in [`PHDestroyable.cpp`](PHDestroyable.cpp.md).

**Needs** — [`Hit.h`](Hit.h.md) · [`PHDestroyableNotificate.h`](PHDestroyableNotificate.h.md)
**Used by** — [`Car.cpp`](Car.cpp.md) · [`Car.h`](Car.h.md) · [`CarSound.cpp`](CarSound.cpp.md) · [`CarWheels.cpp`](CarWheels.cpp.md) · [`CharacterPhysicsSupport.cpp`](CharacterPhysicsSupport.cpp.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`DBG_Car.cpp`](DBG_Car.cpp.md) · [`DestroyablePhysicsObject.cpp`](DestroyablePhysicsObject.cpp.md) · [`DestroyablePhysicsObject.h`](DestroyablePhysicsObject.h.md) · [`Helicopter.cpp`](Helicopter.cpp.md) · [`Helicopter2.cpp`](Helicopter2.cpp.md) · [`Mincer.cpp`](Mincer.cpp.md) · [`Mincer.h`](Mincer.h.md) · [`PHDestroyable.cpp`](PHDestroyable.cpp.md) · _and 4 more_
**Tier floor** — T3: a declaration

## Purpose

Declares `CPHDestroyable` and the `CPHDestroyableNotificator` interface it implements.
Substance is in [`PHDestroyable.cpp`](PHDestroyable.cpp.md).

The shape it fixes: destruction is a **two-party asynchronous handshake**, and the header is
where the two roles are named. A *notificator* is an object that can break; a *notificate*
(declared in [`PHDestroyableNotificate.h`](PHDestroyableNotificate.h.md)) is a piece that can
report back to one. Since the pieces are independently spawned entities, the original cannot
know when they exist except by being told, and it may not remove itself until every one has
told it.

The mixin demands one thing of whoever includes it: a way to reach the physics shell holder
underneath, which is how it gets at the body, the visual and the placement without knowing
what kind of object it is attached to.

Exported units:

- `CPHDestroyableNotificator` — the interface a breakable object presents to its pieces: one
  method, "a piece has arrived".
- `CPHDestroyable` — the mixin: the debris visual list, the pieces that have reported, the
  count still outstanding, the blow that broke it, and three flags.
- `Load` (from the entity's configuration, or from a model's embedded data) — read the debris
  list; an object with none is not destroyable.
- `Init` / `RespawnInit` — the partial and full resets.
- `SetFatalHit` / `FatalHit` — the blow whose momentum the pieces inherit. Held as a value
  because the pieces arrive frames later.
- `Destroy` — begin destruction: spawn one piece per visual and mark the object destroyed.
- `NotificateDestroy` — a piece has arrived; when the last one does, transfer momentum into
  all of them and physically remove the original.
- `GenSpawnReplace` / `InitServerObject` — build and broadcast one piece's server record,
  carrying a back-reference to this object.
- `PhysicallyRemoveSelf` — disable body, collision, visibility and activity, without
  destroying the object.
- `Destroyable` / `Destroyed` / `CanDestroy` — the three state queries.
- `CanRemoveObject` — the implementor's veto over final removal; the base permits it.
- `SheduleUpdate` — the poll that finally destroys the object once every condition holds.
- `m_destroyed_obj_visual_names` — the debris list, public so an implementor can add to it.

## Notes

Two commented-out blocks record a richer momentum-transfer model — a per-destroyable
reference bone and three separate transition factors as member state — that was moved into
model data instead. The configuration comments left behind are the actual documentation for
those keys and are worth reading as such: `imp_transition_factor`, `lv_transition_factor`
and `av_transition_factor` are how much of the blow, the linear velocity and the angular
velocity each fragment inherits.
