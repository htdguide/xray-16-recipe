# src/xrGame/stalker_movement_manager_obstacles_path.cpp

> Finding a path that survives the walk: plan it, simulate walking it against the obstacles it will meet, and replan until the simulated walk finishes.

**Needs** — [`stalker_movement_manager_obstacles.h`](stalker_movement_manager_obstacles.h.md) · [`stalker_movement_manager_space.h`](stalker_movement_manager_space.h.md) · [`restricted_object_obstacle.h`](restricted_object_obstacle.h.md) · [`level_path_manager.h`](level_path_manager.h.md) · [`detail_path_manager.h`](detail_path_manager.h.md) · [`level_path_builder.h`](level_path_builder.h.md) · [`moving_objects.h`](moving_objects.h.md) · [`ai_obstacle.h`](ai_obstacle.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a plan-simulate-replan loop over graph searches

## Purpose

An ordinary path search answers "is there a route through the navigation mesh". That is not
the question a human walking through a populated level needs answered, because the obstacles
along the route are not all known when the search runs — they are discovered *as a function
of where along the path the creature is*.

This file's answer is to **walk the path in simulation before walking it for real**. Plan a
path, then step a virtual creature along it at one-second intervals, querying the obstacle
system at each step exactly as the real creature would. If the simulation reaches the end,
the path is good. If a step discovers an obstacle that changes the mesh, go back and plan
again with that obstacle now known.

That loop is the file. Everything else exists to make it safe: the creature's *current* path
must survive a loop that may fail at any point, so it is saved and restored.

## State

Declared in
[`stalker_movement_manager_obstacles.h`](stalker_movement_manager_obstacles.h.md): the saved
path and its flag, the last destination searched for, and the failure flag shared with the
avoiders.

## `build_level_path`

**Contract** — the whole plan-simulate-replan loop. Replaces the base manager's level
search. Leaves the creature with either a validated path or its previous one. Always
records the destination it ran for.

```text
FUNCTION build_level_path()
  IF obstacle avoidance is disabled THEN base.build_level_path() ; RETURN

  IF the destination has changed since the last search
    forget obstacle records within 5 metres of the creature
  last_fail_time = 0 ; failed_to_build_path = false

  save_current_state()
  # the inactive query becomes a snapshot the loop may mutate freely
  static.inactive_query = copy of static.active_query
  static.inactive_query.update_objects(creature position, unbounded radius)
  dynamic.inactive_query = copy of dynamic.active_query

  pure_search_tried = false ; pure_search_ok = false
  REPEAT
    IF failed_to_build_path THEN BREAK
    base.build_level_path()

    IF the level search failed
      IF NOT pure_search_tried
        pure_search_tried = true
        static.clear() ; saved_query.clear()      # drop every obstacle
        clear the failure record on the level path
        base.build_level_path()                   # try again with no obstacles at all
        pure_search_ok = NOT level_path.failed()
      IF NOT pure_search_ok
        BREAK                                     # genuinely unreachable
  UNTIL simulate_path_navigation()

  last_dest_vertex = level_path.destination
```

**Invariants** — the loop terminates because each iteration either fails outright, succeeds
at the simulation, or records a new obstacle that was not recorded before. The obstacle set
is finite, so the number of replans is bounded by it.

**Notes** — three decisions here matter.

**Forgetting nearby obstacles on a new destination.** When the creature is sent somewhere
new, obstacle records within five metres are dropped. Those records are why the creature is
standing where it is — it stopped or detoured because of them — and carrying them into a
fresh plan would make the new path start with the old detour. Five metres is a feel value;
it is roughly "the obstacles that were affecting me right now".

**The pure search fallback.** If the path search fails *with* obstacles applied, the
obstacle set is discarded entirely and the search is run again. If that succeeds, the loop
continues from there — the creature will walk a path that ignores obstacles and rediscover
them as it goes. This is the choice between "refuse to move" and "move and deal with it",
and the game chooses the second, because a human standing still because a route is
temporarily crowded reads as broken. It is attempted **once** per build; a second failure is
accepted as genuinely unreachable.

**Working against a snapshot.** The loop mutates the *inactive* obstacle query while the
creature's live behaviour continues to read the active one. The two are swapped by the
avoiders when a plan is adopted. Without this, a plan that failed halfway would leave the
creature's live obstacle set half-updated.

## `simulate_path_navigation`

**Contract** — walks a virtual creature along the just-planned detail path, querying the
obstacle system at each step. Reports whether it reached the end. Restores the saved path and
records a failure time if a query says no path is possible at all.

```text
FUNCTION simulate_path_navigation() -> bool
  position = the creature's actual position ; previous = position
  cursor   = 0
  WHILE the detail path is not completed at position
    static.on_before_query()
    static.query(position, previous)             # what obstacles does a creature here meet
    IF NOT static.process_query(allow_replan = false)
      last_fail_time = now ; failed_to_build_path = true
      restore_current_state()
      RETURN false                               # no path exists; keep the old one
    IF static.need_path_to_rebuild()
      RETURN false                               # a new obstacle was learned; replan
    previous = position
    position = predict_position(step_interval, position, cursor, speed = 1)
  RETURN true
```

**Invariants** — the query takes both the current and the *previous* simulated position. An
obstacle is entered and left, and the pair of positions is what tells the obstacle system
which direction the creature is passing through — which side of a door it approaches from,
which way it crosses another creature's path.

The two failure exits are different and must stay different. A query that cannot be
satisfied means *no route exists* and the old path is restored; a query that changed the
mesh means *the plan was made without knowing this* and the caller replans. Conflating them
either loses the creature's path or loops forever.

**Notes** — the simulation steps at **one second of travel at unit speed**, so roughly one
metre. That is the resolution at which obstacles are discovered: something occupying less
than a step's width along the path can be stepped over in simulation and met for real. A
finer step costs proportionally more searches per plan. Neither the interval nor the unit
speed is derived; the unit speed in particular means the simulation walks at one metre per
second regardless of how fast the creature actually moves, so the *spatial* resolution is
what was chosen, not the temporal one.

The simulation reuses the same position prediction the aiming and obstacle systems use (see
[`movement_manager.cpp`](movement_manager.cpp.md)), which is what keeps the simulated walk
and the real one in agreement.

## `save_current_state` / `restore_current_state`

**Contract** — save the creature's entire current path so that a failed replan leaves it
exactly as it was. Saving is conditional: only a path that is complete and actually leads to
the destination the builder is targeting is worth keeping.

```text
FUNCTION save_current_state()
  saved = false
  IF the level path is empty                                  THEN RETURN
  IF the level path does not end at the builder's destination  THEN RETURN
  IF the detail path is empty                                  THEN RETURN
  IF the detail path's destination is not the builder's        THEN RETURN
  saved = true
  take the level path, the detail path, the travel cursor,
       the last patrol point and the obstacle iteration

FUNCTION restore_current_state()
  IF NOT saved THEN RETURN
  put all five back
```

**Invariants** — the four guards together mean "this path is current and goes where we are
going". A path to a stale destination is not worth restoring, so it is not saved and the
failed replan simply leaves the creature with no path — which the pipeline handles.

The saved fields are **exchanged** with the live ones rather than copied, so the storage is
reused on the way back and no allocation happens on this path. That is why saving and
restoring must be exactly paired: the live containers hold the saved contents in between.

**Notes** — the obstacle iteration is saved along with the path because it is the record of
*which obstacles this path was planned around*. Restoring the path without it would leave the
creature walking a route it can no longer explain.

## `remove_query_objects`

**Contract** — forgets obstacle records within a radius of a point, from both the active and
the inactive query. Used only on a change of destination. Both queries must be cleared or
the surviving one re-supplies the records at the next swap.
