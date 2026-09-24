# src/xrGame/ai/monsters/states/monster_state_home_point_danger.h

> Declares the retreat-to-territory composite used by the frightening-sound and shot-from-nowhere
> behaviours, implemented in
> [`monster_state_home_point_danger_inline.h`](monster_state_home_point_danger_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_home_point_danger_inline.h`](monster_state_home_point_danger_inline.h.md)
**Used by** — [`group_state_hear_danger_sound_inline.h`](../group_states/group_state_hear_danger_sound_inline.h.md) · [`monster_state_hear_danger_sound_inline.h`](monster_state_hear_danger_sound_inline.h.md) · [`monster_state_hitted_inline.h`](monster_state_hitted_inline.h.md) · [`monster_state_home_point_danger_inline.h`](monster_state_home_point_danger_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the three-step retreat a frightened territorial creature performs: run to a claimed covered
spot inside its home region, turn to watch the open ground, then hold there.

Its private state is the claimed destination vertex, a flag recording whether that destination was
covered or merely inside the territory, and a scratch vector holding the position of whatever is
currently most frightening.

## `CStateMonsterDangerMoveToHomePoint`

- **construct** — register the three leaves
- **enter** — work out the danger position, claim a destination
- **leave** (clean and forced) — release the claim
- **is_startable** — outside the territory, and the danger is outside it too
- **is_finished** — hit again, or the destination was uncovered and the run is over
- **reselect** / **setup** — the three-step sequence and its parameters
