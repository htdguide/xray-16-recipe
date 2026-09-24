# src/xrGame/ai/monsters/chimera/chimera_state_threaten_walk.h

> Declares the walking approach used during an intimidation display.

**Needs** — [`state_move_to_point.h`](../states/state_move_to_point.h.md) · [`chimera_state_threaten_walk_inline.h`](chimera_state_threaten_walk_inline.h.md)
**Used by** — [`chimera_state_threaten_inline.h`](chimera_state_threaten_inline.h.md) · [`chimera_state_threaten_walk_inline.h`](chimera_state_threaten_walk_inline.h.md)
**Tier floor** — T3: parameter authoring over a shared movement state

## Purpose

Declares the surface implemented in [`chimera_state_threaten_walk_inline.h`](chimera_state_threaten_walk_inline.h.md). Adds no data.

## `ChimeraThreatenWalkState`

Overrides `initialize`, `execute`, `check_completion` and `check_start_conditions`.
