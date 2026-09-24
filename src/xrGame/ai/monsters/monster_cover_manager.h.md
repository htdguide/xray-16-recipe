# src/xrGame/ai/monsters/monster_cover_manager.h

> Declares the creature's two cover queries: find a hiding place relative to a threat, and find the most open direction to face.

**Needs** — [`monster_cover_manager.cpp`](monster_cover_manager.cpp.md)
**Used by** — [`base_monster.cpp`](basemonster/base_monster.cpp.md) · [`base_monster_startup.cpp`](basemonster/base_monster_startup.cpp.md) · [`bloodsucker_attack_state_hide_inline.h`](bloodsucker/bloodsucker_attack_state_hide_inline.h.md) · [`bloodsucker_predator_inline.h`](bloodsucker/bloodsucker_predator_inline.h.md) · [`bloodsucker_predator_lite_inline.h`](bloodsucker/bloodsucker_predator_lite_inline.h.md) · [`controller_state_attack_hide_inline.h`](controller/controller_state_attack_hide_inline.h.md) · [`controller_state_attack_hide_lite_inline.h`](controller/controller_state_attack_hide_lite_inline.h.md) · [`group_state_eat_drag_inline.h`](group_states/group_state_eat_drag_inline.h.md) · [`group_state_home_point_attack_inline.h`](group_states/group_state_home_point_attack_inline.h.md) · [`group_state_rest_idle_inline.h`](group_states/group_state_rest_idle_inline.h.md) · [`monster_cover_manager.cpp`](monster_cover_manager.cpp.md) · [`monster_home.cpp`](monster_home.cpp.md) · [`psy_dog_state_psy_attack_hide_inline.h`](pseudodog/psy_dog_state_psy_attack_hide_inline.h.md) · [`monster_state_attack_camp_inline.h`](states/monster_state_attack_camp_inline.h.md) · _and 6 more_
**Tier floor** — T2: owns a long-lived evaluator object reused across queries

## Purpose

Declares the surface implemented in [`monster_cover_manager.cpp`](monster_cover_manager.cpp.md).

## Exported units

- `load` — builds the evaluator, binding it to the creature's movement restrictions. Must run
  before any query.
- `find_cover(threat_position, min, max, deviation)` — the best cover point, searched around
  the creature's own position.
- `find_cover(search_origin, threat_position, min, max, deviation)` — the same, searched around
  a different origin.
- `least_cover_direction` — a heading pointing at the most exposed nearby direction.
