# src/xrGame/ai/monsters/group_states/group_state_eat_eat.h

> Declares the feeding leaf state, implemented in
> [`group_state_eat_eat_inline.h`](group_state_eat_eat_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`group_state_eat_eat_inline.h`](group_state_eat_eat_inline.h.md)
**Used by** — [`group_state_eat_eat_inline.h`](group_state_eat_eat_inline.h.md) · [`group_state_eat_inline.h`](group_state_eat_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the state in which the creature is actually consuming a corpse — the rung that transfers
nourishment from the body to the creature. Split from its body only because C++ splits templates
that way.

## `CStateGroupEating`

- **enter** — clear the bite clock
- **execute** — play the eating action and transfer one bite per bite interval
- **may_start** — close enough to the nearest gripped bone of the corpse
- **is_finished** — the squad leader came too close, the meal timed out, the claim changed, or the
  creature drifted out of reach
- **remove_links** — forget the corpse when that object is destroyed

Its private state is the corpse it verified against and the timestamp of the last bite.
Contracts are in the implementation twin.
