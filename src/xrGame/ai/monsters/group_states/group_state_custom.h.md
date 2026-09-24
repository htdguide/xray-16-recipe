# src/xrGame/ai/monsters/group_states/group_state_custom.h

> Declares the wrapper that plays one numbered "flavour" animation, implemented in
> [`group_state_custom_inline.h`](group_state_custom_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`group_state_custom_inline.h`](group_state_custom_inline.h.md)
**Used by** — [`group_state_attack_inline.h`](group_state_attack_inline.h.md) · [`group_state_custom_inline.h`](group_state_custom_inline.h.md) · [`group_state_eat_inline.h`](group_state_eat_inline.h.md) · [`group_state_rest_inline.h`](group_state_rest_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the one-substate composite that stands the creature still and plays whichever clip of the
numbered animation vocabulary was most recently requested. Split from its body only because C++
splits templates that way.

## `CStateCustomGroup`

- **construct** — register the single substate, a generic "hold this action" state
- **execute** — select that substate, start the requested clip, run the substate
- **setup_substates** — choose the accompanying sound from the clip number
- **is_finished** — reads the creature's own "the clip is done" flag directly
- **remove_links** — forward the destruction notice to the substate

Contracts are in the implementation twin.
