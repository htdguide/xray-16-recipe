# src/xrGame/ai/monsters/group_states/group_state_eat_drag.h

> Declares the corpse-dragging state, implemented in
> [`group_state_eat_drag_inline.h`](group_state_eat_drag_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`group_state_eat_drag_inline.h`](group_state_eat_drag_inline.h.md)
**Used by** — [`group_state_eat_drag_inline.h`](group_state_eat_drag_inline.h.md) · [`group_state_eat_inline.h`](group_state_eat_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the leaf state in which a creature physically grips a ragdoll and hauls it toward the inner
part of its home region. Split from its body only because C++ splits templates that way.

## `CStateGroupDrag`

- **enter** — work out which bones may be gripped, attempt the physical capture, and choose a
  destination
- **execute** — walk backward in the drag gait toward that destination
- **leave** (clean and forced) — release the grip
- **is_finished** — arrived, or the grip was lost, or the attempt failed outright
- **remove_links** — forward the destruction notice

Its private state is the destination (position plus navigation vertex), a failure flag, and the
corpse's position at the moment the grip was taken. Contracts are in the implementation twin.
