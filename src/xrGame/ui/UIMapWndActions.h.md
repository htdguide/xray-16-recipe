# src/xrGame/ui/UIMapWndActions.h

> Declares the map view's planner: the same goal-driven planner the creatures use, applied to
> the question of how a map view should travel from where it is to where it was asked to go.

**Needs** — [`UIMapWndActions.cpp`](UIMapWndActions.cpp.md) · [`UIMapWndActionsSpace.h`](UIMapWndActionsSpace.h.md) · [`action_planner.h`](../action_planner.h.md) · [`property_evaluator_const.h`](../property_evaluator_const.h.md)
**Used by** — [`UIMapWnd.cpp`](UIMapWnd.cpp.md) · [`UIMapWnd.h`](UIMapWnd.h.md) · [`UIMapWndActions.cpp`](UIMapWndActions.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIMapWndActions.cpp`](UIMapWndActions.cpp.md): one
planner type, specialised to the map screen, configured to run without its own logging.

## Exported units

- **The map action planner** — a planner over the map screen, carrying the evaluators and
  operators the twin describes.
- `setup` — install the evaluators, operators and goal.
- `object_name` — the empty string; the planner's diagnostic label is unused here.
