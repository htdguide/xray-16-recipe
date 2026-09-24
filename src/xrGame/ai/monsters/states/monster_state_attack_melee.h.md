# src/xrGame/ai/monsters/states/monster_state_attack_melee.h

> Declares the bite: stand, face the target, play the attack action, and let the creature's melee checker say when to start and when to stop.

**Needs** — [`monster_state_attack_melee_inline.h`](monster_state_attack_melee_inline.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`controller_state_attack_inline.h`](../controller/controller_state_attack_inline.h.md) · [`group_state_attack_inline.h`](../group_states/group_state_attack_inline.h.md) · [`monster_state_attack_inline.h`](monster_state_attack_inline.h.md) · [`monster_state_attack_melee_inline.h`](monster_state_attack_melee_inline.h.md)
**Tier floor** — T3: a leaf state with no state of its own

## Purpose

Declares the surface implemented in
[`monster_state_attack_melee_inline.h`](monster_state_attack_melee_inline.h.md). Stateless: the
distance hysteresis that decides when a creature engages and disengages lives in the creature's
melee checker, not here, which is why every creature can share one melee state.

## Exported units

- construction and destruction — nothing beyond the base.
- `execute` — one tick of biting.
- `check_start_conditions` — delegated to the melee checker, plus visibility.
- `check_completion` — delegated to the melee checker.
- `remove_links` — forwards to the base cascade.
