# src/xrGame/ai/monsters/states/state_move_to_point.h

> Declares the two go-to-a-place leaf states: a plain one that paths once, and an extended one that re-paths, prefers cover and can demand an arrival facing.

**Needs** — [`state.h`](../state.h.md) · [`state_data.h`](state_data.h.md) · [`state_move_to_point_inline.h`](state_move_to_point_inline.h.md)
**Used by** — [`bloodsucker_attack_state_hide_inline.h`](../bloodsucker/bloodsucker_attack_state_hide_inline.h.md) · [`bloodsucker_attack_state_inline.h`](../bloodsucker/bloodsucker_attack_state_inline.h.md) · [`bloodsucker_predator_inline.h`](../bloodsucker/bloodsucker_predator_inline.h.md) · [`bloodsucker_predator_lite_inline.h`](../bloodsucker/bloodsucker_predator_lite_inline.h.md) · [`chimera_state_threaten_steal.h`](../chimera/chimera_state_threaten_steal.h.md) · [`chimera_state_threaten_steal_inline.h`](../chimera/chimera_state_threaten_steal_inline.h.md) · [`chimera_state_threaten_walk.h`](../chimera/chimera_state_threaten_walk.h.md) · [`chimera_state_threaten_walk_inline.h`](../chimera/chimera_state_threaten_walk_inline.h.md) · [`group_state_eat_inline.h`](../group_states/group_state_eat_inline.h.md) · [`group_state_hear_danger_sound_inline.h`](../group_states/group_state_hear_danger_sound_inline.h.md) · [`group_state_home_point_attack_inline.h`](../group_states/group_state_home_point_attack_inline.h.md) · [`group_state_rest_idle_inline.h`](../group_states/group_state_rest_idle_inline.h.md) · [`monster_state_attack_camp_inline.h`](monster_state_attack_camp_inline.h.md) · [`monster_state_controlled_follow_inline.h`](monster_state_controlled_follow_inline.h.md) · _and 12 more_
**Tier floor** — T3: two declarations

## Purpose

Declares the surface implemented in
[`state_move_to_point_inline.h`](state_move_to_point_inline.h.md). Between them these two
states carry almost all creature locomotion in the game: approaching a corpse, walking a
patrol leg, moving to cover, following a squad leader, closing on a smart-terrain job.

The split into a plain and an extended form is not arbitrary. The plain state commits to
one route at entry and is the cheap option for short, uncontested moves. The extended state
re-paths on a cadence, asks the route search to prefer covered cells, and can require the
creature to end up facing a given way — three things that each cost search work, bundled
together because every caller that wants one wants all three.

## Exported units

- **the plain move-to-point state** — entry, per-tick execution, completion.
- **the extended move-to-point state** — the same three, plus re-path cadence, cover
  preference and destination orientation. It is the base class for two of the chimera's
  stalking states, which is the only place a leaf state is subclassed.
