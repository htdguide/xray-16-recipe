# src/xrGame/DestroyablePhysicsObject.h

> Declares the breakable physics prop implemented in [`DestroyablePhysicsObject.cpp`](DestroyablePhysicsObject.cpp.md).

**Needs** — [`PhysicObject.h`](PhysicObject.h.md) · [`PHDestroyable.h`](PHDestroyable.h.md) · [`PHCollisionDamageReceiver.h`](PHCollisionDamageReceiver.h.md) · [`hit_immunity.h`](hit_immunity.h.md) · [`damage_manager.h`](damage_manager.h.md)
**Used by** — [`DestroyablePhysicsObject.cpp`](DestroyablePhysicsObject.cpp.md) · [`PhysicObject_script.cpp`](PhysicObject_script.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the breakable prop as the composition of five capabilities: a physics object, a
destructible (which owns the swap to a broken model), a receiver of damage from physical
collisions, an immunity table and a per-bone damage scaling table. That composition *is*
the design — no one of the five knows about the others, and this class is the only place
they meet. Substance in
[`DestroyablePhysicsObject.cpp`](DestroyablePhysicsObject.cpp.md).

Exported units:

- `CDestroyablePhysicsObject` — the prop; health, break sound, break particles, attacker
  whitelist.
- `net_Spawn` — configure every damage subsystem from the model's embedded user data.
- `Hit` — script callback, whitelist, immunity, bone scale, base, then health.
- `Destroy` — protected: death callback, model swap, sound, blow-oriented particles,
  re-register for scheduling.
- `InitServerObject` — write the broken state into the authoritative record; debris
  serializes differently from an original.
- `CanRemoveObject` — refuse removal while the break effects are still running.
- `OnChangeVisual` — drop the physics shell before the skeleton changes under it.
- `shedule_Update`, `net_Destroy`, `_construct` — lifecycle.
- The cast accessors by which the engine reaches each of the five capabilities.
