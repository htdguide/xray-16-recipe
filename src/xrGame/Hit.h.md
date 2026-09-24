# src/xrGame/Hit.h

> Declares the damage event record, implemented in [`Hit.cpp`](Hit.cpp.md).

**Needs** — [`alife_space.h`](../xrServerEntities/alife_space.h.md)
**Used by** — [`Actor.cpp`](Actor.cpp.md) · [`CarDoors.cpp`](CarDoors.cpp.md) · [`CarSound.cpp`](CarSound.cpp.md) · [`CarWheels.cpp`](CarWheels.cpp.md) · [`CharacterPhysicsSupport.cpp`](CharacterPhysicsSupport.cpp.md) · [`CustomZone.cpp`](CustomZone.cpp.md) · [`DBG_Car.cpp`](DBG_Car.cpp.md) · [`DestroyablePhysicsObject.cpp`](DestroyablePhysicsObject.cpp.md) · [`GameObject.cpp`](GameObject.cpp.md) · [`GameObject.h`](GameObject.h.md) · [`Hit.cpp`](Hit.cpp.md) · [`Mincer.cpp`](Mincer.cpp.md) · [`PHDestroyable.cpp`](PHDestroyable.cpp.md) · [`PHDestroyable.h`](PHDestroyable.h.md) · _and 7 more_
**Tier floor** — T3: a declaration only

## Purpose

Declares `SHit`, the single record every source of damage produces and every recipient
consumes. Substance — the field meanings, the invariants and the frozen wire order — is in
[`Hit.cpp`](Hit.cpp.md).

Exported units:

- `SHit` — the damage event.
- The full constructor, taking damage, direction, attacker, bone, point in bone space,
  impulse, damage type, armour piercing and the aimed-shot flag; and a default
  constructor producing an explicitly *invalid* hit.
- `is_valide`, `invalidate` — the validity marker that lets "the last hit I took" be a
  plain field rather than an optional.
- `damage`, `direction`, `initiator`, `bone`, `bone_space_position`, `phys_impulse`,
  `type` — the checked accessors.
- `GenHeader` — stamp destination, event type and server time before transmitting.
- `Write_Packet`, `Write_Packet_Cont`, `Read_Packet`, `Read_Packet_Cont` — the wire
  encoding, with and without the event header.
- `_dump` — debug diagnostics.

## Notes

The record's fields are public with the private marker commented out in the original.
Several call sites reach in directly — the wound flag and the weapon identifier in
particular are set after construction rather than through the constructor — so in practice
the whole record is the module's shared vocabulary rather than an encapsulated type.
