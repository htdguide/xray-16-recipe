# src/xrGame/ai/monsters/bloodsucker/bloodsucker_vampire_hide.h

> Declares the two-step retreat a bloodsucker performs after feeding: bolt, then go back to stalking.

**Needs** — [`state.h`](../state.h.md) · [`bloodsucker_vampire_hide_inline.h`](bloodsucker_vampire_hide_inline.h.md)
**Used by** — [`bloodsucker_vampire_hide_inline.h`](bloodsucker_vampire_hide_inline.h.md) · [`bloodsucker_vampire_inline.h`](bloodsucker_vampire_inline.h.md)
**Tier floor** — T2: behaviour over the shared state contract

## Purpose

Declares the surface implemented in [`bloodsucker_vampire_hide_inline.h`](bloodsucker_vampire_hide_inline.h.md). It is a composite state with no state of its own — everything it holds lives in the shared substate machinery — so the declaration is only the list of contract points it overrides.

## `VampireHideState`

A composite state. It overrides substate selection, substate parameter filling and the completion test; it adds no fields.
