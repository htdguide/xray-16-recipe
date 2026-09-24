# src/xrGame/stalker_alife_planner.h

> Declares the stalker's default idle brain as a script-extensible planner that is itself an operator.

**Needs** — [`action_planner_action_script.h`](action_planner_action_script.h.md)
**Used by** — [`stalker_alife_planner.cpp`](stalker_alife_planner.cpp.md) · [`stalker_planner.cpp`](stalker_planner.cpp.md)
**Tier floor** — T3: a declaration over the planner base

## Purpose

Declares the surface implemented in [`stalker_alife_planner.cpp`](stalker_alife_planner.cpp.md).

## Exported units

- `setup` — bind to a stalker and rebuild the planning problem.
- `add_evaluators` — install the three property evaluators.
- `add_actions` — install the three operators with their preconditions and effects.

## Notes

The base it derives from supplies three things at once: planning, the operator lifecycle (so
this planner can be installed inside a larger plan), and the script registration that lets a
mod add or replace operators by identifier. That stacking is why the type declaration is
three lines and the behaviour is a page.
