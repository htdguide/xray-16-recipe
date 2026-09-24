# src/xrGame/ai/monsters/burer/burer_fast_gravi.h

> Declares a close-range instant gravity strike, wired into the burer's control slots and never activated.

**Needs** — [`control_combase.h`](../control_combase.h.md) · [`burer_fast_gravi.cpp`](burer_fast_gravi.cpp.md)
**Used by** — [`burer.cpp`](burer.cpp.md) · [`burer_fast_gravi.cpp`](burer_fast_gravi.cpp.md)
**Tier floor** — T3: a control ability declaration

## Purpose

Declares the surface implemented in [`burer_fast_gravi.cpp`](burer_fast_gravi.cpp.md). It is a *control ability*, not a behaviour state — a component the creature's control manager can activate, which then borrows the creature for as long as it needs and hands it back. Abilities sit below states: a state decides what to do, an ability performs something that needs to own the creature's body.

## `BurerFastGravi`

A custom control ability with no data of its own. It fills in the ability contract's four points — start conditions, activation, deactivation, and the event hook — and one private step that applies the damage.
