# src/xrGame/ai/monsters/monster_home.h

> Declares the home area: the place a creature belongs to, as a set of nested radii around an authored patrol path or a single vertex, and the queries that pick a destination inside it.

**Needs** — [`monster_home.cpp`](monster_home.cpp.md)
**Used by** — [`ai_monster_squad_attack.cpp`](ai_monster_squad_attack.cpp.md) · [`base_monster.cpp`](basemonster/base_monster.cpp.md) · [`base_monster_debug.cpp`](basemonster/base_monster_debug.cpp.md) · [`base_monster_startup.cpp`](basemonster/base_monster_startup.cpp.md) · [`bloodsucker_attack_state_hide_inline.h`](bloodsucker/bloodsucker_attack_state_hide_inline.h.md) · [`bloodsucker_attack_state_inline.h`](bloodsucker/bloodsucker_attack_state_inline.h.md) · [`bloodsucker_predator_inline.h`](bloodsucker/bloodsucker_predator_inline.h.md) · [`bloodsucker_predator_lite_inline.h`](bloodsucker/bloodsucker_predator_lite_inline.h.md) · [`dog.cpp`](dog/dog.cpp.md) · [`dog_state_manager.cpp`](dog/dog_state_manager.cpp.md) · [`group_state_eat_drag_inline.h`](group_states/group_state_eat_drag_inline.h.md) · [`group_state_hear_danger_sound_inline.h`](group_states/group_state_hear_danger_sound_inline.h.md) · [`group_state_home_point_attack_inline.h`](group_states/group_state_home_point_attack_inline.h.md) · [`group_state_panic_run_inline.h`](group_states/group_state_panic_run_inline.h.md) · _and 13 more_
**Tier floor** — T3: radii, random picks, and navigation-graph lookups

## Purpose

Declares the surface implemented in [`monster_home.cpp`](monster_home.cpp.md).

## Exported units

- `load(line)` — reads the home definition from the creature's spawn configuration.
- `setup(path_name, …)` / `setup(vertex, …)` — establish a home from script, around an authored
  patrol path or a single navigation vertex.
- `clear_home` — forget it.
- `set_wander_step(min, max)` — the distance band of one idle move.
- `place_in_inner`, `place_in_middle`, `place_in_outer`, `place_towards`, `place`,
  `place_in_cover` — six ways to choose a destination vertex inside the home.
- `contains()` / `contains(position)` / `contains(position, radius)`, `within_inner`,
  `within_middle` — containment against the three radii.
- `anchor_position` — one representative point of the home.
- `inner_radius`, `middle_radius`, `outer_radius`, `has_home`, `is_aggressive`.
