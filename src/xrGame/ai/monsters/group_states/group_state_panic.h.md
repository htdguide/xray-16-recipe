# src/xrGame/ai/monsters/group_states/group_state_panic.h

> Declares the pack panic brain, implemented in
> [`group_state_panic_inline.h`](group_state_panic_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`group_state_panic_inline.h`](group_state_panic_inline.h.md)
**Used by** — [`dog_state_manager.cpp`](../dog/dog_state_manager.cpp.md) · [`group_state_panic_inline.h`](group_state_panic_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the composite a pack creature enters when it is afraid of a known enemy. Split from its body
only because C++ splits templates that way.

## `CStateGroupPanic`

- **construct** — register three substates: flee, face the open ground, and go home
- **enter** — nothing beyond the base contract
- **reselect_state** — go home if needed, otherwise alternate fleeing and facing
- **check_force_state** — the two interrupts that cut a pause short
- **setup_substates** — parameterise the facing rung
- **remove_links** — forward the destruction notice

Contracts are in the implementation twin.
