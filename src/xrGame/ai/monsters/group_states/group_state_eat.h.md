# src/xrGame/ai/monsters/group_states/group_state_eat.h

> Declares the pack feeding brain, implemented in
> [`group_state_eat_inline.h`](group_state_eat_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`group_state_eat_inline.h`](group_state_eat_inline.h.md)
**Used by** — [`dog_state_manager.cpp`](../dog/dog_state_manager.cpp.md) · [`group_state_eat_inline.h`](group_state_eat_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the composite that takes a pack creature from "there is a corpse" through approaching,
dragging it somewhere private, feeding, and withdrawing. Split from its body only because C++
splits templates that way.

## `CStateGroupEat`

- **reset** — clear the satiety clock
- **enter** — latch the corpse the creature claimed
- **reselect_state** — the sequencer over eight substates
- **setup_substates** — fill in the parameters of the five data-driven substates
- **leave** (clean and forced) — release the corpse, drop the physical capture, unlock the corpse
  for other creatures with a timeout
- **is_finished** — the corpse changed, or we stopped being hungry
- **may_start** — we already hold a corpse, or one is available, inside the home region, unlocked,
  and we are hungry
- **remove_links** — forget the latched corpse when that object is destroyed

Its private state is the latched corpse and the last-fed timestamp. Contracts are in the
implementation twin.
