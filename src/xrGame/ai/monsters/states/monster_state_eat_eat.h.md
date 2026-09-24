# src/xrGame/ai/monsters/states/monster_state_eat_eat.h

> Declares the actual feeding leaf, implemented in
> [`monster_state_eat_eat_inline.h`](monster_state_eat_eat_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_eat_eat_inline.h`](monster_state_eat_eat_inline.h.md)
**Used by** — [`monster_state_eat_eat_inline.h`](monster_state_eat_eat_inline.h.md) · [`monster_state_eat_inline.h`](monster_state_eat_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the leaf in which the creature stands at a corpse and takes bites out of it, transferring
food from the body's reserve into the creature's satiety. Carries one compiled-in number, the
twenty-second cap on a single sitting.

## `CStateMonsterEating`

- **enter** — reset the bite clock
- **execute** — play the eating action and transfer one slice per interval
- **is_startable** — is the creature close enough to the nearest part of the body
- **is_finished** — twenty seconds elapsed, the corpse changed, or the creature drifted away

Its private state is the corpse it latched onto at start-condition time and the timestamp of the
last bite. Contracts are in the implementation twin.
