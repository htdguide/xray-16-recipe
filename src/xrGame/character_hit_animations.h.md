# src/xrGame/character_hit_animations.h

> Declares the hit-reaction controller implemented in [`character_hit_animations.cpp`](character_hit_animations.cpp.md).

**Needs** — [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md) · [`entity_alive.h`](entity_alive.h.md)
**Used by** — [`CharacterPhysicsSupport.cpp`](CharacterPhysicsSupport.cpp.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`character_hit_animations.cpp`](character_hit_animations.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in
[`character_hit_animations.cpp`](character_hit_animations.cpp.md). Embedded by value in a
living entity; nine motion handles, nine blend slots and a bone index.

Exported units:

- **`SetupHitMotions`** — bind the nine motion names and the spine bone to a model.
- **`PlayHitMotion`** — layer the reactions for one hit.
- **`GetBaseMatrix`** — the spine bone's world transform, the frame reactions are computed in.
- **`IsEffected`** (private) — whether a struck bone is under the spine.

**Notes** — the blend slots are mutable so the reaction can be played through a read-only
reference to the controller. That the controller mutates while nominally const is worth
knowing: the blocking rule is state, and a rebuild that makes the play operation read-only
loses it.
