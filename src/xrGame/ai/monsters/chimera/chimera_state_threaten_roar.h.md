# src/xrGame/ai/monsters/chimera/chimera_state_threaten_roar.h

> Declares the leaf that stands still and bellows at the target.

**Needs** — [`state.h`](../state.h.md) · [`chimera_state_threaten_roar_inline.h`](chimera_state_threaten_roar_inline.h.md)
**Used by** — [`chimera_state_threaten_inline.h`](chimera_state_threaten_inline.h.md) · [`chimera_state_threaten_roar_inline.h`](chimera_state_threaten_roar_inline.h.md)
**Tier floor** — T3: a leaf state

## Purpose

Declares the surface implemented in [`chimera_state_threaten_roar_inline.h`](chimera_state_threaten_roar_inline.h.md). Carries no data of its own; it reads the entry time the shared state machinery records for it.

## `ChimeraThreatenRoarState`

A leaf state overriding `initialize`, `execute` and `check_completion`.
