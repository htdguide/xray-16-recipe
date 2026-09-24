# src/xrGame/ai/monsters/states/monster_state_steal.h

> Declares the stalking-approach leaf, implemented in
> [`monster_state_steal_inline.h`](monster_state_steal_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_steal_inline.h`](monster_state_steal_inline.h.md)
**Used by** — [`monster_state_attack_inline.h`](monster_state_attack_inline.h.md) · [`monster_state_steal_inline.h`](monster_state_steal_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the combat leaf in which a creature closes on an enemy who has not noticed it, creeping
instead of charging. Registered by the generic attack behaviour and tested before every other
combat option, so it is the first thing a creature considers when it acquires an enemy.

It carries three compiled-in numbers: the four-to-fifteen-unit band the creeping is allowed in, and
a maximum path deviation that is **dead** — declared, and its only use commented out.

## `CStateMonsterSteal`

- **enter** — prepare the path builder
- **execute** — creep toward the enemy
- **is_startable** / **is_finished** — the same seven-clause predicate, negated

Its start and completion tests are literally each other's negation, which is the interesting part.
Contracts are in the implementation twin.
