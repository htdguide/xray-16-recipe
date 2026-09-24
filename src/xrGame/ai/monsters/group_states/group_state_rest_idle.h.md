# src/xrGame/ai/monsters/group_states/group_state_rest_idle.h

> Declares the wandering-within-the-territory state, implemented in
> [`group_state_rest_idle_inline.h`](group_state_rest_idle_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`group_state_rest_idle_inline.h`](group_state_rest_idle_inline.h.md)
**Used by** — [`group_state_rest_idle_inline.h`](group_state_rest_idle_inline.h.md) · [`group_state_rest_inline.h`](group_state_rest_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the composite a pack creature runs when it has nothing to do and is not asleep: a loop of
walking to a reserved spot, playing an idle animation, and walking somewhere else. Split from its
body only because C++ splits templates that way.

## `CStateGroupRestIdle`

- **construct** — register four substates: walk to the reserved spot, look at the open ground,
  walk to a wander point, and play a numbered flavour animation
- **enter** — choose and reserve a destination
- **leave** (clean and forced) — release the reservation
- **reselect_state** — the three-beat loop
- **setup_substates** — fill in the parameters of the four substates, including the sniffing-gait
  rule
- **remove_links** — forward the destruction notice

Its private state is the reserved vertex and the gait chosen for the current walk. Contracts are
in the implementation twin.
