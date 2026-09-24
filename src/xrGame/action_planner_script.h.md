# src/xrGame/action_planner_script.h

> Declares the bridge that lets an engine-written brain be a script-facing planner. Behaviour is in [`action_planner_script_inline.h`](action_planner_script_inline.h.md).

**Needs** — [`action_planner.h`](action_planner.h.md) · [`action_planner_script_inline.h`](action_planner_script_inline.h.md)
**Used by** — [`action_planner_script_inline.h`](action_planner_script_inline.h.md) · [`smart_cover_animation_planner.h`](smart_cover_animation_planner.h.md) · [`stalker_planner.h`](stalker_planner.h.md)
**Tier floor** — T3: a declaration

## Purpose

The third member of the bridge family, for the planner itself: a brain written in the
engine whose actions and evaluators are the script-facing kind, holding both the game
object facade its parts see and the concrete creature its own code wants.

Substance is in [`action_planner_script_inline.h`](action_planner_script_inline.h.md).

Exported units:

- `setup` — bind the concrete creature, deriving the facade from it.
- `object` — the concrete creature.
