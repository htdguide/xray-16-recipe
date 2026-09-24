# src/xrGame/ai/monsters/group_states/group_state_home_point_attack.h

> Declares an unused pack variant of "fall back to the home region during a fight", implemented in
> [`group_state_home_point_attack_inline.h`](group_state_home_point_attack_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`group_state_home_point_attack_inline.h`](group_state_home_point_attack_inline.h.md)
**Used by** — [`group_state_attack_inline.h`](group_state_attack_inline.h.md) · [`group_state_home_point_attack_inline.h`](group_state_home_point_attack_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names a composite that would take a pack creature back to a cover spot in its territory when the
enemy is unreachable, and have it watch the open ground from there. **Nothing instantiates it** —
the pack attack brain includes this header but registers the generic creature version of the same
idea instead (see [`group_state_attack_inline.h`](group_state_attack_inline.h.md)). Read it as an
alternative that was written and not adopted.

## `CStateGroupAttackMoveToHomePoint`

- **construct** — register two substates: run to a cover spot, and look at the most open direction
- **enter** — clear the inaccessibility clocks and stamp the entry time
- **leave** (clean and forced) — release the squad's reservation on the chosen cover spot
- **may_start** — outside the territory, or the enemy has been unreachable for long enough
- **is_finished** — several ways, described in the implementation twin
- **reselect_state** — alternate between running and looking
- **setup_substates** — choose the cover spot and the look direction
- **enemy_inaccessible** — the four-part reachability test

Its private state is the chosen cover vertex, a flag that suppresses the looking rung, and three
timestamps. Contracts are in the implementation twin.
