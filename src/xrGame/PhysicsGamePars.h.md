# src/xrGame/PhysicsGamePars.h

> Declares the collision-effect thresholds and volumes defined in [`PhysicsGamePars.cpp`](PhysicsGamePars.cpp.md).

**Needs** — _(none)_
**Used by** — [`PhysicsGamePars.cpp`](PhysicsGamePars.cpp.md) · [`physics_game.cpp`](physics_game.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Names the two threshold sets — one for props, one for characters — and the two global
collision-sound gains, all defined in [`PhysicsGamePars.cpp`](PhysicsGamePars.cpp.md).

Exported units:

- `collide_volume_min`, `collide_volume_max` — the gain range for a collision sound.
- `EffectPars` — the prop thresholds for sound, particles and wallmark.
- `CharacterEffectPars` — the same three, raised for character bodies.
- `LoadPhysicsGameParams` — reads the two gains from configuration at startup.
