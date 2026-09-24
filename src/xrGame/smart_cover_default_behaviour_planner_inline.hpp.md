# src/xrGame/smart_cover_default_behaviour_planner_inline.hpp

> The two unused dwell-interval accessors on the default behaviour planner.

**Needs** — [`smart_cover_default_behaviour_planner.hpp`](smart_cover_default_behaviour_planner.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: accessors

## Purpose

Carries the bodies for
[`smart_cover_default_behaviour_planner.hpp`](smart_cover_default_behaviour_planner.hpp.md).

## `idle_time` / `lookout_time`

**Contract** — read and write two stored millisecond intervals. Nothing in the engine calls
either. The live dwell state is on the animation planner
([`smart_cover_animation_planner_inline.h`](smart_cover_animation_planner_inline.h.md));
these are a parallel set that was never wired up, and a rebuild should omit them.
