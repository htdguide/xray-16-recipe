# src/xrGame/ai/monsters/ai_monster_squad_manager.h

> Declares the registry that owns every creature pack on the level and finds a creature's pack from the identity it already carries.

**Needs** — [`ai_monster_squad_manager.cpp`](ai_monster_squad_manager.cpp.md) · [`ai_monster_squad_manager_inline.h`](ai_monster_squad_manager_inline.h.md) · [`ai_monster_squad.h`](ai_monster_squad.h.md)
**Used by** — [`ai_monster_squad_manager.cpp`](ai_monster_squad_manager.cpp.md) · [`ai_monster_squad_manager_inline.h`](ai_monster_squad_manager_inline.h.md) · [`base_monster.cpp`](basemonster/base_monster.cpp.md) · [`base_monster_startup.cpp`](basemonster/base_monster_startup.cpp.md) · [`base_monster_think.cpp`](basemonster/base_monster_think.cpp.md) · [`bloodsucker_attack_state_hide_inline.h`](bloodsucker/bloodsucker_attack_state_hide_inline.h.md) · [`bloodsucker_attack_state_inline.h`](bloodsucker/bloodsucker_attack_state_inline.h.md) · [`bloodsucker_predator_inline.h`](bloodsucker/bloodsucker_predator_inline.h.md) · [`bloodsucker_predator_lite_inline.h`](bloodsucker/bloodsucker_predator_lite_inline.h.md) · [`bloodsucker_state_manager.cpp`](bloodsucker/bloodsucker_state_manager.cpp.md) · [`dog.cpp`](dog/dog.cpp.md) · [`dog_state_manager.cpp`](dog/dog_state_manager.cpp.md) · [`group_state_attack_inline.h`](group_states/group_state_attack_inline.h.md) · [`group_state_attack_run_inline.h`](group_states/group_state_attack_run_inline.h.md) · _and 15 more_
**Tier floor** — T3: a three-level nested sequence indexed by three small integers

## Purpose

Declares the surface implemented in
[`ai_monster_squad_manager.cpp`](ai_monster_squad_manager.cpp.md).

Worth reading for one thing: the declaration records that this registry names its three
levels **team, level, squad** while the rest of the engine names the same three **team,
squad, group**. The nesting is identical and only the words differ. A rebuild should use
one naming; the engine-wide one is the right choice, since it is the one that appears in
spawn data and in every other registry.

## Exported units

- **the pack registry** — register and remove a member by the (team, squad, group) triple,
  look a pack up by triple or by entity, drive one entity's coordination pass, and drop
  references to a destroyed object across every pack.
- **the global accessor** — reaches the one registry, creating it on first use. See
  [`ai_monster_squad_manager_inline.h`](ai_monster_squad_manager_inline.h.md).
