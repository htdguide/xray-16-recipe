# src/xrGame/PHSoundPlayer.h

> Declares the per-object collision-sound gate, implemented in [`PHSoundPlayer.cpp`](PHSoundPlayer.cpp.md).

**Needs** — [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md)
**Used by** — [`CharacterPhysicsSupport.cpp`](CharacterPhysicsSupport.cpp.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`PHSoundPlayer.cpp`](PHSoundPlayer.cpp.md) · [`physics_game.cpp`](physics_game.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CPHSoundPlayer`. Substance is in [`PHSoundPlayer.cpp`](PHSoundPlayer.cpp.md).

The shape it fixes in one field: **one sound handle**. An object gets one collision sound at
a time, and that single handle is the entire throttling mechanism for an otherwise unbounded
stream of contact events.

Exported units:

- `CPHSoundPlayer` — bound to one physical object for its lifetime.
- `Play` — play a collision sound for a material pair at a position, unless one is already
  playing or the object is barely moving.
