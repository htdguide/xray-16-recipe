# src/xrGame/ai/monsters/states/monster_state_squad_rest_follow.h

> Declares the follow-the-squad-order behaviour, implemented in
> [`monster_state_squad_rest_follow_inline.h`](monster_state_squad_rest_follow_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_squad_rest_follow_inline.h`](monster_state_squad_rest_follow_inline.h.md)
**Used by** — [`monster_state_rest_inline.h`](monster_state_rest_inline.h.md) · [`monster_state_squad_rest_follow_inline.h`](monster_state_squad_rest_follow_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names what a pack member does in peacetime when the squad leader's standing order is *follow*:
walk to the position the order names, pausing whenever it is close enough. This is how a moving
pack keeps formation without every member pathing to the leader.

It carries four compiled-in numbers: the two-unit stopping distance, the ten units beyond which it
must move, and the two-to-three-second pause. A declared field holding the last commanded position
is **dead** — written on entry and never read. An overridden pre-emption hook is likewise empty.

## `CStateMonsterSquadRestFollow`

- **enter** — snapshot the commanded position (unused)
- **reselect** — pause if close enough, otherwise walk
- **setup** — fill each leaf's parameters
