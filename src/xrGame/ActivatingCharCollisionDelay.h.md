# src/xrGame/ActivatingCharCollisionDelay.h

> Declares the capsule-creation retry timer implemented in [`ActivatingCharCollisionDelay.cpp`](ActivatingCharCollisionDelay.cpp.md).

**Needs** — [`ActivatingCharCollisionDelay.cpp`](ActivatingCharCollisionDelay.cpp.md)
**Used by** — [`ActivatingCharCollisionDelay.cpp`](ActivatingCharCollisionDelay.cpp.md) · [`CharacterPhysicsSupport.cpp`](CharacterPhysicsSupport.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the small helper that retries creating a creature's upright collision capsule
every three seconds until the creature is standing somewhere free. Substance is in
[`ActivatingCharCollisionDelay.cpp`](ActivatingCharCollisionDelay.cpp.md).

Exported units:

- `activating_character_delay` — the retry object; non-copyable, holds a non-owning
  reference to the creature's physics support and the timestamp of its next attempt.
- `update` — one tick of the retry.
- `active` — whether the capsule is still missing.
- The retry period, three seconds, is fixed here rather than configured.
