# src/xrGame/ai/monsters/states/state_look_point.h

> Declares the turn-to-face leaf state: rotate toward a point and, by default, finish when the turn is done.

**Needs** — [`state.h`](../state.h.md) · [`state_data.h`](state_data.h.md) · [`state_look_point_inline.h`](state_look_point_inline.h.md)
**Used by** — [`bloodsucker_predator_inline.h`](../bloodsucker/bloodsucker_predator_inline.h.md) · [`bloodsucker_predator_lite_inline.h`](../bloodsucker/bloodsucker_predator_lite_inline.h.md) · [`burer_state_attack_inline.h`](../burer/burer_state_attack_inline.h.md) · [`group_state_home_point_attack_inline.h`](../group_states/group_state_home_point_attack_inline.h.md) · [`group_state_rest_idle_inline.h`](../group_states/group_state_rest_idle_inline.h.md) · [`monster_state_attack_camp_inline.h`](monster_state_attack_camp_inline.h.md) · [`monster_state_find_enemy_look_inline.h`](monster_state_find_enemy_look_inline.h.md) · [`monster_state_home_point_danger_inline.h`](monster_state_home_point_danger_inline.h.md) · [`monster_state_rest_idle_inline.h`](monster_state_rest_idle_inline.h.md) · [`state_look_point_inline.h`](state_look_point_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in
[`state_look_point_inline.h`](state_look_point_inline.h.md). Widely used: it is the "look
at an open place" step of the resting and panic behaviours, the "face the enemy" step of
the burer's attack, and the turn inside the enemy-search sweep.

## Exported units

- **the turn-to-point state** — entry, per-tick execution, and a completion test with two
  modes (timeout, or turn-finished).
