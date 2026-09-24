# src/xrGame/static_obstacles_avoider.cpp

> Keeps a creature's route honest about the doors, crates and other creatures standing in
> it — and refuses to adopt an obstacle set that would leave it with nowhere to go.

**Needs** — [`static_obstacles_avoider.h`](static_obstacles_avoider.h.md) · [`obstacles_query.h`](obstacles_query.h.md) · [`refreshable_obstacles_query.h`](refreshable_obstacles_query.h.md) · [`stalker_movement_manager_obstacles.h`](stalker_movement_manager_obstacles.h.md) · [`moving_objects.h`](moving_objects.h.md) · [`moving_object.h`](moving_object.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`static_obstacles_avoider.h`](static_obstacles_avoider.h.md)
**Tier floor** — T2: set algebra over navigation-mesh regions, re-run per creature per cycle.

## Purpose

The level's navigation mesh is static, but the world on top of it is not: objects move, and
a creature walking a route must treat the cells they occupy as unwalkable. The pathfinder
takes such exclusions as part of its cost model — but excluding too much makes *every* route
fail, and a creature with no route does nothing at all, which is worse than a creature that
clips a crate.

This module is the arbiter of that trade. It gathers the obstacles near a creature's route
each cycle and decides whether to adopt them, and it will **decline an obstacle set that
makes the creature's destination unreachable**. That refusal is the load-bearing decision in
the file; everything else is the bookkeeping that makes it cheap.

## State

Four obstacle sets, and the difference between them is the whole design.

```text
RECORD StaticObstaclesAvoider
  current_iteration : ObstaclesQuery     # what the world reported this cycle
  last_iteration    : ObstaclesQuery     # what it reported the previous cycle
  inactive_query    : RefreshableQuery   # the candidate: everything accumulated so far
  active_query      : RefreshableQuery   # what the pathfinder is actually using
  need_path_rebuild : bool               # set when this cycle changed the active set
  path_failed       : reference to bool  # the movement layer's "I could not build a path"
```

**Invariants**

- `active_query` is always a set the creature *can* build a path against. It is only ever
  replaced by a set that has been tested. This is what the whole module guarantees.
- `inactive_query` is the accumulating candidate and may be untestable; the two are equal
  whenever the candidate has been accepted.
- `current_iteration` and `last_iteration` are swapped rather than copied each cycle, so the
  previous cycle's answer is retained for a change test at no cost.
- Each set carries both a set of obstacle objects and the mesh region they cover, and both
  halves participate in every comparison.

## `update`

**Contract** — one cycle: refresh what is nearby, and decide what the pathfinder should see.
Three steps in a fixed order.

```text
FUNCTION update()
  on_before_query()     # retire this cycle's answer to last_iteration, clear, lower the flag
  query()               # ask the moving-objects registry what is near me
  process_query(true)   # decide
```

## `query`

**Contract** — asks the world's moving-object registry which moving objects affect this
creature, and takes ownership of the answer. Two forms: one for a creature in general, one
aimed at a specific start and destination — the latter used when a route is being planned,
so the query is restricted to the corridor the creature actually intends to walk.

**Notes** — the answer is *swapped* out of the registry rather than copied. The registry
computed it for this creature and has no further use for it; moving it avoids a per-cycle
allocation on a path walked by every creature every cycle. A rebuild without cheap ownership
transfer should pool the sets instead of allocating.

## `new_obstacles_found`

**Contract** — is this cycle's answer worth acting on?

```text
FUNCTION new_obstacles_found() -> bool
  IF current_iteration.obstacles IS empty        RETURN false
  IF current_iteration.area IS empty             RETURN false
  IF current_iteration != last_iteration         RETURN true
  RETURN path_failed        # unchanged, but the creature is stuck: try again anyway
```

**Invariants** — the last line is the interesting one. An unchanged obstacle set normally
means there is nothing to do, but if the movement layer is *currently failing to build a
path*, the same obstacles are reprocessed. Without it a creature that became stuck between
two cycles would never re-examine its situation and would stand still until something else
moved.

The two emptiness tests are separate because a set can name obstacles that cover no mesh
cells (an object above or below the walkable surface). Such a set changes nothing for the
pathfinder and must not trigger the work below.

## `process_query`

**Contract** — merge this cycle's obstacles into the candidate, test the result, and adopt
it only if the creature can still move. Returns whether the creature's movement state is
sound. Takes a flag saying whether it is allowed to change the path state at all — a caller
mid-way through committing a route passes false, and the function then only reports.

```text
FUNCTION process_query(may_change_path) -> bool
  IF NOT new_obstacles_found()
    RETURN may_change_path ? refresh_objects() : true

  active_was_current := may_change_path ? (active_query == inactive_query) : true

  merged := inactive_query.merge(my_position,
                                 may_change_path ? inactive_query.refresh_radius : 0,
                                 current_iteration)
  IF NOT merged
    # nothing new after merging; keep the active set in step if it already was
    IF active_was_current
      active_query.copy(inactive_query)
    RETURN may_change_path ? refresh_objects() : true

  IF NOT movement_manager.can_build_restricted_path(inactive_query)
    RETURN may_change_path ? refresh_objects() : false     # candidate rejected

  active_query.copy(inactive_query)
  need_path_rebuild := true
  RETURN true
```

**Invariants** — `can_build_restricted_path` is the gate, and it is asked of the *candidate*
before the candidate becomes active. A candidate that would strand the creature is left
inactive and the creature keeps walking against the older, looser set. It will clip the new
obstacle — and that is the deliberate choice: a creature that walks through a crate is a
visual flaw, a creature that stands still because every route is blocked is a broken game.

The radius passed to the merge is zeroed when the caller forbids path changes, which
restricts the merge to obstacles exactly on the route rather than in a neighbourhood around
the creature.

## `refresh_objects`

**Contract** — re-examine the *active* set for objects that have moved since it was adopted,
and either keep the refreshed set or roll back to the one before it. Returns whether the
creature's movement state is sound.

```text
FUNCTION refresh_objects() -> bool
  radius := active_query.refresh_radius
  IF NOT active_query.objects_changed(my_position, radius)
    RETURN true                              # nothing moved; nothing to do

  saved := copy of active_query              # rollback point
  IF NOT active_query.refresh_objects()
    RETURN true                              # refresh produced no change

  IF NOT movement_manager.can_build_restricted_path(active_query)
    active_query := saved                    # roll back: the refreshed set strands us
    IF inactive_query == active_query
      inactive_query.refresh_objects()       # keep the candidate in step with the rollback
    RETURN false

  inactive_query.update_objects(my_position, radius)
  need_path_rebuild := true
  RETURN true
```

**Invariants** — this is the same refusal as in `process_query`, applied to a set that is
*already* active. Obstacles move; a set that was passable when adopted can become
impassable without any new obstacle appearing. The explicit save-and-restore is what makes
the active set's invariant hold across that: it is never left in a state that has not been
tested.

The conditional refresh of the candidate exists so that the two sets do not silently
diverge when they were equal — otherwise a later equality test would compare a rolled-back
active set against a refreshed candidate and conclude they differ when nothing had changed.

## `on_before_query`

**Contract** — start of cycle: retire this cycle's set into the previous-cycle slot, empty
it, and lower the rebuild flag. The flag is per-cycle, so a caller reads it after `update`
and before the next one.

## `remove_links`

**Contract** — an object is being destroyed; drop it from all four sets. Must reach every
set, including the retired previous-cycle one, because that one is compared against next
cycle and a destroyed object left in it would make an unchanged world look changed.

## `clear`

**Contract** — empties all four sets. Used when a creature's movement is reset — a level
change, a teleport — where nothing about the previous position's obstacles applies.

## Accessors

**Contract** — `need_path_to_rebuild`, `active_query`, `inactive_query` and
`current_iteration` expose the sets to the movement manager, which reads the flag each cycle
to decide whether to re-run the pathfinder and reads the active set to hand to it.
