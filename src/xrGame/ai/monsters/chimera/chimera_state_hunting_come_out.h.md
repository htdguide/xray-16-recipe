# src/xrGame/ai/monsters/chimera/chimera_state_hunting_come_out.h

> Declares the "emerge from cover" half of the unbuilt hunting behaviour.

**Needs** — [`state.h`](../state.h.md) · [`chimera_state_hunting_come_out_inline.h`](chimera_state_hunting_come_out_inline.h.md)
**Used by** — [`chimera_state_hunting_come_out_inline.h`](chimera_state_hunting_come_out_inline.h.md) · [`chimera_state_hunting_inline.h`](chimera_state_hunting_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented — or rather, not implemented — in [`chimera_state_hunting_come_out_inline.h`](chimera_state_hunting_come_out_inline.h.md). Part of the unbuilt subtree described in [`chimera_state_hunting_inline.h`](chimera_state_hunting_inline.h.md).

## `ChimeraHuntingComeOutState`

A leaf state overriding substate selection and the two predicates. Carries no data. Note that it declares *substate selection* despite being a leaf with no substates, which is one sign that the file was copied from its sibling and never finished.
