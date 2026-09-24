# src/xrGame/stalker_planner_inline.h

> The stalker planner's one flag accessor.

**Needs** — [`stalker_planner.h`](stalker_planner.h.md)
**Used by** — [`stalker_planner.h`](stalker_planner.h.md)
**Tier floor** — T3: a boolean field.

## Purpose

Split out of the header for compilation reasons only; substance is in
[`stalker_planner.cpp`](stalker_planner.cpp.md).

## `affect_cover`

**Contract** — get and set a boolean saying whether this creature's current activity should
be allowed to influence the squad's cover bookkeeping. Set false on every planner setup,
so its default is "do not affect"; anything that wants the opposite must raise it
deliberately.
