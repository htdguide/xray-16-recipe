# src/xrGame/ai/monsters/states/monster_state_squad_rest.h

> Declares the loiter-near-the-leader behaviour, implemented in
> [`monster_state_squad_rest_inline.h`](monster_state_squad_rest_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_squad_rest_inline.h`](monster_state_squad_rest_inline.h.md)
**Used by** — [`monster_state_rest_inline.h`](monster_state_rest_inline.h.md) · [`monster_state_squad_rest_inline.h`](monster_state_squad_rest_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names what a pack member does in peacetime when its squad leader has ordered the pack to rest:
alternate at random between standing still and ambling to a random spot within twenty units of the
leader. It is the behaviour that makes a resting pack look like a pack rather than like several
animals that happen to be nearby.

It carries four compiled-in numbers: the five-to-ten-second idle window, the twenty-unit radius
around the leader, and the five attempts allowed when looking for a spot inside it. A declared
field for the next reselection time is **dead** — never written, never read.

## `CStateMonsterSquadRest`

- **construct** — register two leaves: stand still, and amble near the leader
- **reselect** — choose between them by coin flip
- **setup** — fill each leaf's parameters
