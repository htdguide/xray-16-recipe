# src/xrGame/ai/monsters/states/state_hide_from_point.h

> Declares the retreat leaf state: get away from a position, using cover, until told to stop.

**Needs** — [`state.h`](../state.h.md) · [`state_data.h`](state_data.h.md) · [`state_hide_from_point_inline.h`](state_hide_from_point_inline.h.md)
**Used by** — [`bloodsucker_vampire_hide_inline.h`](../bloodsucker/bloodsucker_vampire_hide_inline.h.md) · [`bloodsucker_vampire_inline.h`](../bloodsucker/bloodsucker_vampire_inline.h.md) · [`group_state_attack_inline.h`](../group_states/group_state_attack_inline.h.md) · [`group_state_eat_inline.h`](../group_states/group_state_eat_inline.h.md) · [`monster_state_attack_inline.h`](monster_state_attack_inline.h.md) · [`monster_state_eat_inline.h`](monster_state_eat_inline.h.md) · [`monster_state_hear_danger_sound_inline.h`](monster_state_hear_danger_sound_inline.h.md) · [`state_hide_from_point_inline.h`](state_hide_from_point_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in
[`state_hide_from_point_inline.h`](state_hide_from_point_inline.h.md). The state owns one
`StateHideFromPoint` record, which the composite state above it fills before selecting it.

## Exported units

- **the retreat state** — entry (prepares the path builder), per-tick execution, and a
  completion test. It declares no start condition, so it can always be selected.

**Notes** — this is one of the most widely reused leaf states in the creature layer: the
blood-sucker's escape after a feed, the group attack's break-off, the panic reaction to a
dangerous sound and the eating behaviour's withdrawal from a corpse all resolve to it.
