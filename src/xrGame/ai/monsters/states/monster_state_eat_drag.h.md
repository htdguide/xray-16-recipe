# src/xrGame/ai/monsters/states/monster_state_eat_drag.h

> Declares the solitary creature's corpse-dragging leaf, implemented in
> [`monster_state_eat_drag_inline.h`](monster_state_eat_drag_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_eat_drag_inline.h`](monster_state_eat_drag_inline.h.md)
**Used by** — [`monster_state_eat_drag_inline.h`](monster_state_eat_drag_inline.h.md) · [`monster_state_eat_inline.h`](monster_state_eat_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the leaf in which a creature grips a corpse and hauls it backward to a covered spot before
settling down to eat. Split from its body only because the original splits templates that way.

## `CStateMonsterDrag`

- **enter** — take a physical grip on the corpse and choose a covered destination
- **execute** — back away toward that destination in the dragging gait
- **leave** (clean and forced) — drop the corpse
- **is_finished** — arrived, or the grip was lost, or the grip was never taken

Its private state is the destination (position plus navigation vertex), a failure flag, and the
corpse's position at the moment the grip was taken. Contracts are in the implementation twin.
