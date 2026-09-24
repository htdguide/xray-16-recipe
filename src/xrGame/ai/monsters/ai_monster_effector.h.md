# src/xrGame/ai/monsters/ai_monster_effector.h

> Declares the two screen effects a creature inflicts on the player: a post-process wash with an attack/hold/release envelope, and a decaying camera shake.

**Needs** — [`ActorEffector.h`](../../ActorEffector.h.md) · [`ai_monster_effector.cpp`](ai_monster_effector.cpp.md)
**Used by** — [`ai_monster_effector.cpp`](ai_monster_effector.cpp.md) · [`base_monster_feel.cpp`](basemonster/base_monster_feel.cpp.md) · [`controller.cpp`](controller/controller.cpp.md) · [`poltergeist_flame_thrower.cpp`](poltergeist/poltergeist_flame_thrower.cpp.md) · [`psy_dog.cpp`](pseudodog/psy_dog.cpp.md) · [`pseudo_gigant.cpp`](pseudogigant/pseudo_gigant.cpp.md) · [`scanning_ability_inline.h`](scanning_ability_inline.h.md)
**Tier floor** — T2: two per-frame interpolators feeding the camera and post-process chains

## Purpose

Declares the surface implemented in [`ai_monster_effector.cpp`](ai_monster_effector.cpp.md).

## Exported units

- **creature post-process effector** — blends a configured post-process state in and out
  over a lifetime, with separately tuned attack and release fractions and an overall
  strength factor.
- **creature hit camera effector** — a three-axis oscillation of the camera's orientation
  whose amplitude decays to nothing over its lifetime.
