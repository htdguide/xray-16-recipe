# src/xrGame/ai/monsters/states/monster_state_attack_on_run.h

> Declares the circling attack: for creatures that fight while moving, a state that never ends — it orbits the enemy, predicts where the enemy will be, and strikes in passing from alternating sides.

**Needs** — [`monster_state_attack_on_run_inline.h`](monster_state_attack_on_run_inline.h.md) · [`../state.h`](../state.h.md) · [`../../weighted_random.h`](../../weighted_random.h.md)
**Used by** — [`group_state_attack_inline.h`](../group_states/group_state_attack_inline.h.md) · [`monster_state_attack_inline.h`](monster_state_attack_inline.h.md) · [`monster_state_attack_on_run_inline.h`](monster_state_attack_on_run_inline.h.md)
**Tier floor** — T2: predicts a target's position from measured velocity and solves for an intercept point each tick

## Purpose

Declares the surface implemented in
[`monster_state_attack_on_run_inline.h`](monster_state_attack_on_run_inline.h.md). This is the
largest single behaviour in chapter 24 and the only one that replaces the whole approach/melee
chain rather than sitting inside it: a creature whose `can_attack_on_move` predicate is true
never reaches the classic branches at all.

## State

```text
RECORD AttackOnRunState
  # the movement phase
  phaze                   : ENUM { go_close, go_far, go_prepare }
  phaze_chosen_time       : int
  go_far_start_point      : vector          # where the current disengage began
  # which side to pass on
  attack_side             : ENUM { left, right }
  attack_side_chosen_time : int
  prepare_side            : ENUM { left, right }
  prepare_side_chosen_time: int
  # the strike
  attacking               : bool
  attack_end_time         : int              # when the strike animation finishes
  enemy_to_attack         : optional<entity> # may differ from the primary enemy
  animation_index[2]      : int              # the chosen variant per side
  animation_hit_time[2]   : real             # seconds into that variant at which the hit lands
  # the destination
  target                  : vector
  target_vertex           : int
  reach_old_target        : bool             # finish the current pass before re-planning
  reach_old_target_start  : int
  # enemy prediction
  last_prediction_time    : int
  last_update_enemy_pos   : vector
  predicted_enemy_velocity: vector
  predicted_enemy_pos     : vector
  # miscellaneous
  can_do_rotation_jump    : bool             # re-rolled on each disengage
  is_jumping              : bool             # edge-detected against the creature
  try_min_time            : bool             # computed and never used; see the implementation
  try_min_time_period     : int              # likewise
  try_min_time_chosen_time: int              # likewise
  can_do_preparation      : bool             # declared and never used
  last_update_time        : int              # assigned once and never read
```

**Invariants** — `phaze` is declared with three values and **only two are ever used**:
`go_close` and `go_prepare`. Nothing assigns `go_far`, and a namespace constant naming a maximum
duration for it sits unused beside the two that are used. A rebuild should drop it.

Five further fields are inert: the three min-time fields are maintained by a routine whose
result is discarded, and two booleans are declared and never touched. They are listed above so
a reader diffing against the source is not left wondering.

## Exported units

- construction, `initialize`, `execute`, `finalize`, `critical_finalize`.
- `check_start_conditions` — the same window as the charge.
- `check_completion` — always false.
- `remove_links` — must drop the secondary attack target as well as cascading.
- `check_control_start_conditions` — the veto that gates the rotation jump and the anti-aim step.
- Private: the prediction, the destination solve, the side choice, the strike, the phase change,
  the animation choice, and the fallback destination scan.

**Notes** — the file also declares a free helper taking one argument that is **defined nowhere**;
the implementation defines a two-argument function of the same name instead. The one-argument
declaration is dangling and unreferenced.
