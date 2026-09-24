# src/xrGame/ai/monsters/cat/cat.h

> Declares the cat: the base creature with a pounce that the shipped build no longer arms.

**Needs** — [`base_monster.h`](../basemonster/base_monster.h.md) · [`cat.cpp`](cat.cpp.md)
**Used by** — [`cat.cpp`](cat.cpp.md) · [`cat_script.cpp`](cat_script.cpp.md) · [`cat_state_manager.cpp`](cat_state_manager.cpp.md)
**Tier floor** — T2: a creature class

## Purpose

Declares the surface implemented in [`cat.cpp`](cat.cpp.md). The cat adds nothing to the base creature that survives into the shipped build: its distinguishing ability, a jump attack, is declared and half-wired and then left disabled.

## `Cat`

Beyond the base lifecycle points (`Load`, `reinit`, `UpdateCL`, `CheckSpecParams`) it declares:

- `try_to_jump` — the pounce trigger, reduced to a pair of guards and no action.
- `HitEntityInJump` — the damage a landed pounce would deal, read from the attack-parameters table for the pounce's third clip.
- `get_monster_class_name` — returns `"cat"`.
- script registration — see [`cat_script.cpp`](cat_script.cpp.md).
