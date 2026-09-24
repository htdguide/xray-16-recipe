# src/xrGame/ai/monsters/chimera/chimera_state_threaten_steal.h

> Declares the creeping approach used during an intimidation display.

**Needs** — [`state_move_to_point.h`](../states/state_move_to_point.h.md) · [`chimera_state_threaten_steal_inline.h`](chimera_state_threaten_steal_inline.h.md)
**Used by** — [`chimera_state_threaten_inline.h`](chimera_state_threaten_inline.h.md) · [`chimera_state_threaten_steal_inline.h`](chimera_state_threaten_steal_inline.h.md)
**Tier floor** — T3: parameter authoring over a shared movement state

## Purpose

Declares the surface implemented in [`chimera_state_threaten_steal_inline.h`](chimera_state_threaten_steal_inline.h.md). It derives from the shared "move to a point, re-targeting as it goes" state and only fills in that state's parameter record, so it adds no data.

## `ChimeraThreatenStealState`

Overrides `initialize`, `execute`, `finalize`, `check_completion` and `check_start_conditions`.
