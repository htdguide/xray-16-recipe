# src/xrGame/ai/monsters/ai_monster_bones.h

> Declares the additive bone-rotation layer that lets a creature aim or flinch a body part independently of whatever animation is playing.

**Needs** — [`ai_monster_bones.cpp`](ai_monster_bones.cpp.md)
**Used by** — [`ai_monster_bones.cpp`](ai_monster_bones.cpp.md) · [`bloodsucker.cpp`](bloodsucker/bloodsucker.cpp.md) · [`bloodsucker.h`](bloodsucker/bloodsucker.h.md) · [`controller_direction.cpp`](controller/controller_direction.cpp.md) · [`controller_direction.h`](controller/controller_direction.h.md) · [`zombie.cpp`](zombie/zombie.cpp.md) · [`zombie.h`](zombie/zombie.h.md)
**Tier floor** — T2: rotation accumulation applied inside the animation layer's per-bone callback

## Purpose

Declares the surface implemented in [`ai_monster_bones.cpp`](ai_monster_bones.cpp.md).

## Exported units

- **axis selector** — a three-bit set naming which of the model's local axes a given
  rotation applies about. A bone may be registered more than once with different axis sets;
  each registration is an independent channel.
- **per-bone rotation channel** — current angle, target angle, turn rate, and the angular
  distance the current move started from. Knows how to decide whether it still needs to
  turn, to advance one step, and to compose itself onto the bone's transform.
- **the manipulation set** — the collection of channels for one creature, the shared
  hold-then-return timer, and the two queries (is anything active, is it returning) the
  behaviour code branches on.
