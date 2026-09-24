# src/xrGame/ai/monsters/bloodsucker/bloodsucker_predator.h

> Declares the full stalking loop a bloodsucker falls into after feeding: take cover, face the open ground, and wait.

**Needs** — [`state.h`](../state.h.md) · [`bloodsucker_predator_inline.h`](bloodsucker_predator_inline.h.md)
**Used by** — [`bloodsucker_predator_inline.h`](bloodsucker_predator_inline.h.md) · [`bloodsucker_state_manager.cpp`](bloodsucker_state_manager.cpp.md) · [`bloodsucker_vampire_hide_inline.h`](bloodsucker_vampire_hide_inline.h.md)
**Tier floor** — T3: a declaration over the shared state contract

## Purpose

Declares the surface implemented in [`bloodsucker_predator_inline.h`](bloodsucker_predator_inline.h.md). This is the reachable predator state — the retreat tree of the vampire behaviour selects it; see [`bloodsucker_vampire_hide_inline.h`](bloodsucker_vampire_hide_inline.h.md). Its lighter sibling [`bloodsucker_predator_lite.h`](bloodsucker_predator_lite.h.md) is a different state with a different selector, not a variant of this one.

## `BloodsuckerPredatorState`

A composite state holding the navigation vertex it has claimed from its pack and the tick its current camp began. It overrides entry, substate selection, both exits, the start and completion tests, the parameter fill and the forced-restart hook.
