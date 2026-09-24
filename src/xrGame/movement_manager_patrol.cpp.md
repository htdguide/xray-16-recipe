# src/xrGame/movement_manager_patrol.cpp

> Drives the path pipeline when the destination comes from an authored patrol route: pick the next point, walk to it, pick the next.

**Needs** — [`movement_manager.h`](movement_manager.h.md) · [`patrol_path_manager.h`](patrol_path_manager.h.md) · [`level_path_manager.h`](level_path_manager.h.md) · [`detail_path_manager.h`](detail_path_manager.h.md) · [`level_path_builder.h`](level_path_builder.h.md) · [`detail_path_builder.h`](detail_path_builder.h.md) · [`CustomMonster.h`](CustomMonster.h.md) · [`mt_config.h`](mt_config.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: graph search orchestration

## Purpose

The third pipeline driver. A patrol path is an authored graph of waypoints shipped with the
level; this driver asks the patrol path manager which waypoint comes next, then runs the
ordinary level-and-detail search to reach it, and repeats. It is how most of the world's
idle population appears to be going somewhere.

## State

`Stateless.`

## `process_patrol_path`

**Contract** — advances the path state machine by one step for the patrol-path kind.
Non-blocking; defers searches to the worker pool when permitted.

```text
FUNCTION process_patrol_path()
  IF level path stale AND state past build_level_path      THEN state := build_level_path
  IF patrol stale     AND state past select_patrol_point   THEN state := select_patrol_point

  STEP select_patrol_point
    ask the patrol manager for the next waypoint, seeded from our position;
      it writes the chosen level vertex straight into the level path's destination
    IF the patrol manager failed THEN BREAK
    IF the route is finished     THEN state := completed; BREAK
    fall through

  STEP build_level_path
    configure the level builder with (our level vertex, the destination vertex,
      the patrol manager's extrapolation choice, the waypoint's exact position)
    IF deferrable THEN queue it; BREAK
    run inline; BREAK

  STEP continue_level_path
    pick the next intermediate vertex; fall through

  STEP build_detail_path
    patrol-style iff the patrol manager says extrapolate
    start position := our position; start heading := negated body yaw
    destination := the waypoint's exact position
    configure the detail builder; queue or run; BREAK

  STEP verification
    patrol stale        -> select_patrol_point
    level path stale    -> build_level_path
    detail path stale   -> build_level_path
    detail path consumed -> continue_level_path,
      and if the level path is consumed -> select_patrol_point,
      and if the route is finished      -> completed

  STEP completed
    patrol stale -> select_patrol_point
```

**Invariants** — the staleness cascade has an extra rung compared to the level driver:
patrol staleness rewinds further than level staleness, which rewinds further than detail
staleness. The order of the two pre-switch checks matters — the level check runs first and
the patrol check second, so a simultaneously-stale pair lands on the *earlier* stage.

Completing a level path returns to point selection rather than to completion: reaching one
waypoint is not reaching the end of the route. Only the patrol manager reporting the route
finished ends the pipeline.

**Notes** — the waypoint's exact position is supplied to both the level builder and the
detail manager, because a waypoint is an authored world point that need not coincide with
any navigation vertex. The level search reaches the vertex; the detail path reaches the
authored point. A rebuild that snaps waypoints to vertices will visibly change patrol
behaviour on the shipped data, where waypoints are placed against level art.

This file is compiled into the script-aware translation unit, unlike its two siblings. That
is an artifact of how the patrol path manager reaches the script layer for its
waypoint-selection callbacks and carries no meaning for a rebuild.
