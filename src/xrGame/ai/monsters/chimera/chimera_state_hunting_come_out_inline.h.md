# src/xrGame/ai/monsters/chimera/chimera_state_hunting_come_out_inline.h

> Nothing: this file defines the wrong state, and is the reason the hunting subtree cannot be built.

**Needs** — [`chimera_state_hunting_come_out.h`](chimera_state_hunting_come_out.h.md)
**Used by** — [`chimera_state_hunting_come_out.h`](chimera_state_hunting_come_out.h.md)
**Tier floor** — T3: nothing to implement

## Purpose

This file is supposed to implement the "emerge from cover" state declared in [`chimera_state_hunting_come_out.h`](chimera_state_hunting_come_out.h.md). It does not. It is a verbatim copy of its sibling [`chimera_state_hunting_move_to_cover_inline.h`](chimera_state_hunting_move_to_cover_inline.h.md), defining the *move to cover* state a second time — including a substate-selection body that names the parent's two substates, which are not in scope here at all.

The consequence is structural rather than cosmetic: pulling the hunting subtree into any translation unit defines the same state twice and fails to define the one that was asked for. That is the proof that nothing in the shipped engine ever includes it, and it is why [`chimera_state_hunting_inline.h`](chimera_state_hunting_inline.h.md) is described as a design note rather than as code.

A rebuild carries nothing forward from this file. What the *intended* state had to do is recorded on the parent's twin.

## State

Stateless.
