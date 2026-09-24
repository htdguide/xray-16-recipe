# src/xrGame/ai/monsters/states/monster_state_home_point_rest.h

> Declares the wander-back-to-the-middle-of-the-territory leaf, implemented in
> [`monster_state_home_point_rest_inline.h`](monster_state_home_point_rest_inline.h.md).

**Needs** — [`monster_state_move.h`](monster_state_move.h.md) · [`monster_state_home_point_rest_inline.h`](monster_state_home_point_rest_inline.h.md)
**Used by** — [`group_state_rest_inline.h`](../group_states/group_state_rest_inline.h.md) · [`monster_state_home_point_rest_inline.h`](monster_state_home_point_rest_inline.h.md) · [`monster_state_rest_inline.h`](monster_state_rest_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the leaf that returns an idle creature to the inner part of its home region. It is the calm
counterpart of the two danger retreats — nothing is threatening the creature, it has simply
drifted, and this walks it back.

It derives from the movement base in [`monster_state_move.h`](monster_state_move.h.md), which
exists purely to make sure the path builder is prepared on entry. Its private state is the
destination vertex, chosen once.

## `CStateMonsterRestMoveToHomePoint`

- **enter** — choose a cell in the territory's inner region
- **execute** — go there, at a gait chosen by the territory's temperament
- **is_startable** — the creature is outside the inner region
- **is_finished** — standing on the destination cell with no path left
