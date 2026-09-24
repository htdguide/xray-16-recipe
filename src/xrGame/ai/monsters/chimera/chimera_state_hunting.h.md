# src/xrGame/ai/monsters/chimera/chimera_state_hunting.h

> Declares an unfinished hunting behaviour: take cover, then come out.

**Needs** — [`state.h`](../state.h.md) · [`chimera_state_hunting_inline.h`](chimera_state_hunting_inline.h.md)
**Used by** — [`chimera_state_hunting_inline.h`](chimera_state_hunting_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`chimera_state_hunting_inline.h`](chimera_state_hunting_inline.h.md). Neither this state nor its two substates has a working implementation, and nothing instantiates them; see that twin.

## `ChimeraHuntingState`

A composite state with two substate slots, *move to cover* and *come out*. It overrides substate selection and the two start/finish predicates, and carries no data.
