# src/xrGame/ai/monsters/states/monster_state_rest_sleep.h

> Declares the sleeping leaf, implemented in
> [`monster_state_rest_sleep_inline.h`](monster_state_rest_sleep_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_rest_sleep_inline.h`](monster_state_rest_sleep_inline.h.md) · [`../../../ai_debug.h`](../../../ai_debug.h.md)
**Used by** — [`group_state_rest_inline.h`](../group_states/group_state_rest_inline.h.md) · [`monster_state_rest_inline.h`](monster_state_rest_inline.h.md) · [`monster_state_rest_sleep_inline.h`](monster_state_rest_sleep_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the leaf in which a creature sleeps. Registered by both peacetime composites but selected by
only one of them: the pack variant
([`../group_states/group_state_rest_inline.h`](../group_states/group_state_rest_inline.h.md))
chooses it on a schedule, while the solitary variant
([`monster_state_rest_inline.h`](monster_state_rest_inline.h.md)) registers it and never names it.

Stateless. Its whole substance is that entry and exit are not symmetric with anything else in this
directory: they change the creature's perception, not just its animation.

## `CStateMonsterRestSleep`

- **enter** — put the creature to sleep
- **execute** — play the sleeping action
- **leave** (clean and forced) — wake the creature
