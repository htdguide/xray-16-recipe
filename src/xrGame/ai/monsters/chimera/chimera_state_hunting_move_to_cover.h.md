# src/xrGame/ai/monsters/chimera/chimera_state_hunting_move_to_cover.h

> Declares the "get into cover" half of the unbuilt hunting behaviour.

**Needs** — [`state.h`](../state.h.md) · [`chimera_state_hunting_move_to_cover_inline.h`](chimera_state_hunting_move_to_cover_inline.h.md)
**Used by** — [`chimera_state_hunting_inline.h`](chimera_state_hunting_inline.h.md) · [`chimera_state_hunting_move_to_cover_inline.h`](chimera_state_hunting_move_to_cover_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`chimera_state_hunting_move_to_cover_inline.h`](chimera_state_hunting_move_to_cover_inline.h.md). Part of the unbuilt subtree described in [`chimera_state_hunting_inline.h`](chimera_state_hunting_inline.h.md).

## `ChimeraHuntingMoveToCoverState`

A leaf state overriding `initialize`, `execute` and the two predicates. Carries no data.
