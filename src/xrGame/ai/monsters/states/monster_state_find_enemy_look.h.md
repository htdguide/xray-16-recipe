# src/xrGame/ai/monsters/states/monster_state_find_enemy_look.h

> Declares the casting-about leaf of the lost-contact search, implemented in
> [`monster_state_find_enemy_look_inline.h`](monster_state_find_enemy_look_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_find_enemy_look_inline.h`](monster_state_find_enemy_look_inline.h.md)
**Used by** — [`monster_state_find_enemy_inline.h`](monster_state_find_enemy_inline.h.md) · [`monster_state_find_enemy_look_inline.h`](monster_state_find_enemy_look_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the composite that makes a creature sweep the area around where it lost contact: alternating
look-arounds with 120-degree turns and short dashes, five steps, then done.

Its private state is the chosen sweep handedness, a step counter, the current facing, the position
it entered at, and the point currently being turned or run toward. It also declares three local
step identifiers that shadow the shared state vocabulary with a private enumeration — dead
declarations: the actual leaves are registered under the shared identifiers from
[`../state_defs.h`](../state_defs.h.md) and the private enumeration is never read.

## `CStateMonsterFindEnemyLook`

- **enter** — pick a handedness at random and reset the step counter
- **reselect** — alternate between looking around and turning or dashing to a side
- **setup** — fill the parameters of whichever step was chosen
- **is_finished** — after five steps
