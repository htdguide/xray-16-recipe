# src/xrGame/ai/monsters/group_states/group_state_rest.h

> Declares the pack idling brain, implemented in
> [`group_state_rest_inline.h`](group_state_rest_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`group_state_rest_inline.h`](group_state_rest_inline.h.md)
**Used by** — [`dog_state_manager.cpp`](../dog/dog_state_manager.cpp.md) · [`group_state_rest_inline.h`](group_state_rest_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the composite that runs whenever a pack creature has nothing else to do — which, for most
creatures on a level, is nearly all the time. Split from its body only because C++ splits templates
that way.

## `CStateGroupRest`

- **construct** — register six substates: sleep, move into a restricted region, go home, perform a
  smart-terrain job, idle in the territory, and play a numbered flavour animation
- **enter** — draw this waking period's length and arm the anomaly detector
- **execute** — the priority ladder and the sleep cycle
- **leave** (clean and forced) — disarm the anomaly detector

Its private state is two deadlines: when the creature may next sleep, and when it must next wake.
Contracts are in the implementation twin.
