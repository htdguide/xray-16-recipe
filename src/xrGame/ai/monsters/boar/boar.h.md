# src/xrGame/ai/monsters/boar/boar.h

> Declares the boar: a base creature plus a head that tracks the enemy and a jump-turn.

**Needs** — [`base_monster.h`](../basemonster/base_monster.h.md) · [`controlled_entity.h`](../controlled_entity.h.md) · [`boar.cpp`](boar.cpp.md)
**Used by** — [`boar.cpp`](boar.cpp.md) · [`boar_script.cpp`](boar_script.cpp.md) · [`boar_state_manager.cpp`](boar_state_manager.cpp.md)
**Tier floor** — T2: a creature class, no device or format contact

## Purpose

Declares the surface implemented in [`boar.cpp`](boar.cpp.md). The boar is the base creature with two additions: it can be taken over by a controller creature (it mixes in the "controllable" role), and its head yaws independently of its body toward whatever it is fighting.

## `Boar`

The creature class. Beyond the base creature's lifecycle points (`Load`, `net_Spawn`, `reinit`, `UpdateCL`, `CheckSpecParams`) it exposes:

- `look_at_enemy`, `current_head_delta`, `target_head_delta`, `head_turn_speed` — the head-tracking state, public because the bone callback reaches in from outside the object.
- `BoneCallback` — the per-frame hook the animation layer calls to post-multiply the head bone's transform.
- `CanExecRotationJump` — answers yes; the base creature asks before using the jump-turn.
- `ability_can_drag` — answers yes; the boar can drag corpses.
- `get_monster_class_name` — returns `"boar"`, the key configuration and scripts address it by.
- script registration — see [`boar_script.cpp`](boar_script.cpp.md).
