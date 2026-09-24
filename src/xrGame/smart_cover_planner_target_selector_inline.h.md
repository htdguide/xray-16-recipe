# src/xrGame/smart_cover_planner_target_selector_inline.h

> The read accessor for the target selector's script callback.

**Needs** — [`smart_cover_planner_target_selector.h`](smart_cover_planner_target_selector.h.md)
**Used by** — [`smart_cover_planner_target_selector.h`](smart_cover_planner_target_selector.h.md)
**Tier floor** — T4: one field read

## Purpose

Exists only because the original separates inline bodies from declarations so that a
header can be included without pulling the definitions into every translation unit. That
is a C++ compilation concern and disappears in a rebuild; fold the accessor into the type.

## State

`Stateless.`

## `callback`

**Contract** — returns the currently installed script callback of the target selector. No
side effects.
