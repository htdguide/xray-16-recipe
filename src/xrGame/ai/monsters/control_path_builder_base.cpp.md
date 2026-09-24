# src/xrGame/ai/monsters/control_path_builder_base.cpp

> Lifecycle and event handling for the path-builder base: it subscribes to the navigation stack's three events and turns them into the failure and end-of-path facts the per-frame pass reads.

**Needs** — [`control_path_builder_base.h`](control_path_builder_base.h.md) · [`control_path_builder.h`](control_path_builder.h.md) · [`cover_evaluators.h`](../../cover_evaluators.h.md) · [`level_path_manager.h`](../../level_path_manager.h.md) · [`detail_path_manager.h`](../../detail_path_manager.h.md) · [`level_location_selector.h`](../../level_location_selector.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../../../xrAICore/Navigation/ai_object_location.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: event handling on the frame path

## Purpose

The part of the path-builder base that listens. Three events come from the path resource —
a path was built, a path was updated, the creature passed a waypoint — and this file turns
them into the two facts the per-frame state pass consumes: *did something fail*, and
*have we reached the end*. It also owns reset and the cover evaluator.

## State

Operates on the record in [`control_path_builder_base.h`](control_path_builder_base.h.md).
Owns the cover evaluator outright, constructed lazily on first reset against the
creature's restriction set.

## `reinit` and `reset`

**Contract** — create the cover evaluator if it does not exist, reset every parameter to
its default, and **capture the path channel**. Reset leaves movement disabled, minimum-time
preference off, all gaits admissible, no destination orientation, the level path type, no
game-graph target, and both timestamps cleared; it then runs the builder preparation from
[`control_path_builder_base_set.cpp`](control_path_builder_base_set.cpp.md), which resets
the target pair.

**Notes** — the cover evaluator is created once and reused for the creature's lifetime,
because it holds a reference to the restriction set. Recreating it on every reset would be
correct and slower; the lazy construction is the only reason `reinit` is not simply
`reset`.

## `on_start_control` / `on_stop_control`

**Contract** — on taking the path channel, subscribe to the three navigation events; on
losing it, unsubscribe. So an ability that seizes the path silences this element
completely — it stops receiving events as well as stopping its writes.

## `on_event`

**Contract** — route path-built to `on_path_built`, path-updated to `on_path_updated`, and
waypoint-change to `travel_point_changed`.

## `on_path_built`

**Contract** — a fresh path exists, so clear the end-of-path flag — but only if the path is
non-empty and the creature is not already standing on its last waypoint.

## `on_path_updated`

**Contract** — inspect the navigation stack after an update pass and decide whether
anything failed. Three independent failures, any of which sets the flag:

```text
FUNCTION on_path_updated()
  IF the level path search reported failure THEN
    failed <- true
    reset the level path search        # so it will try again next time

  IF the detailed path reported failure THEN
    failed <- true

  # a path that is up to date, movement enabled, and yet has no forward waypoints,
  # while we are not standing on the target and the target is still current,
  # means the navigation stack has quietly given up
  IF (no path OR standing on the last waypoint)
     AND the detailed path is up to date
     AND movement is enabled
     AND target_set.node IS NOT my current vertex
     AND target_actual THEN
    failed <- true

  time_path_updated_external <- now
```

**Notes** — the third condition is the load-bearing one and it has no equivalent anywhere
else. The navigation stack does not report "I planned a path of length zero to somewhere
you are not"; it reports success with an empty path. This condition detects exactly that,
and without it a creature whose target is unreachable stands still forever with everything
apparently working. A rebuild that reports the failure honestly from the search can drop
the condition.

Resetting the level path search after observing its failure is not tidying: the search
latches its failure flag, and leaving it latched would make every subsequent update report
a failure too.

## `travel_point_changed` / `on_path_end`

**Contract** — when the creature passes a waypoint, check whether it was the last, and if
so record that the path has ended.

## `global_failed`

**Contract** — are we within three seconds of the last observed failure. While true, target
resolution abandons the requested target and picks random reachable points instead, and
the *actuality* of the current target is ignored so a new target is chosen every pass.

**Notes** — three seconds is a constant in this file. It is the creature's visible "I am
stuck, let me try elsewhere" window, and it is short enough that a creature caught on
geometry looks confused for a moment rather than broken.

## `set_dest_direction`

**Contract** — set the facing the creature should arrive with, rate-limited by the same
rebuild interval that gates replanning. A request arriving too soon is dropped.

## `set_target_accessible`

**Contract** — given a position, produce a (position, node) pair the creature may actually
go to. A position already inside the creature's restrictions is taken as-is with an
unknown node; an inaccessible one is replaced by the nearest accessible position and its
vertex.

**Notes** — the asymmetry is deliberate. An accessible position keeps an *unknown* node so
that node resolution can happen later and cheaply; only the corrected case pays for a
vertex up front, because the correction produced one anyway.

## `detour_graph_points`

**Contract** — switch to the cross-level game path and aim at a game-graph vertex, or at
none, in which case the creature wanders the graph. This is the entry point the alife
layer uses to move a creature between levels.
