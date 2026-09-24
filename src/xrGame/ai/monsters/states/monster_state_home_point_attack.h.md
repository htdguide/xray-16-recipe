# src/xrGame/ai/monsters/states/monster_state_home_point_attack.h

> Declares the fall-back-to-territory leaf used during combat and panic, implemented in
> [`monster_state_home_point_attack_inline.h`](monster_state_home_point_attack_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_home_point_attack_inline.h`](monster_state_home_point_attack_inline.h.md)
**Used by** — [`controller_state_attack_inline.h`](../controller/controller_state_attack_inline.h.md) · [`group_state_attack_inline.h`](../group_states/group_state_attack_inline.h.md) · [`group_state_panic_inline.h`](../group_states/group_state_panic_inline.h.md) · [`monster_state_attack_inline.h`](monster_state_attack_inline.h.md) · [`monster_state_home_point_attack_inline.h`](monster_state_home_point_attack_inline.h.md) · [`monster_state_panic_inline.h`](monster_state_panic_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the leaf that pulls a fighting creature back inside its authored home region, hopping from
one claimed spot to the next. Registered by the attack behaviours, the panic behaviours and
several species-specific attack trees.

Its private state is the currently claimed destination — a navigation vertex and its position — and
the time that destination was chosen. A declared `skip_camp` flag is **dead**: it is never written
and never read.

## `CStateMonsterAttackMoveToHomePoint`

- **enter** — claim a first destination
- **execute** — re-claim when arrived or when the last attempt found nothing; run there
- **leave** (clean and forced) — release the claim
- **is_startable** — outside the territory, or the enemy is unreachable and the species cares
- **is_finished** — back inside the territory, with the same unreachability caveat
