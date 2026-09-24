# src/xrGame/ai/monsters/control_path_builder_base_path.cpp

> Target resolution: turning a position the state layer asked for into a mesh vertex the creature can actually path to, with a four-stage escalation and a random-wander fallback.

**Needs** — [`control_path_builder_base.h`](control_path_builder_base.h.md) · [`control_path_builder.h`](control_path_builder.h.md) · [`cover_point.h`](../../cover_point.h.md) · [`cover_manager.h`](../../cover_manager.h.md) · [`cover_evaluators.h`](../../cover_evaluators.h.md) · [`level_path_manager.h`](../../level_path_manager.h.md) · [`detail_path_manager.h`](../../detail_path_manager.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../../../xrAICore/Navigation/ai_object_location.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: mesh walks, a cover query and a bounded graph search

## Purpose

The most consequential file in the path-builder base. A state layer names a position; this
file produces the (position, vertex) pair the navigation stack will be asked to reach. The
position may be unreachable, forbidden, inside geometry, or the creature's own feet, and
the resolution is an escalation from cheap tests to expensive ones, with a deliberate
give-up path at the end.

Two distances govern everything here and both are constants in this file: the *scatter
distance* of thirty world units, which is how far a random alternative is thrown, and the
*attempt count* of five, which bounds every random retry.

## State

Writes `target_found` on the record in
[`control_path_builder_base.h`](control_path_builder_base.h.md).

## `target_point_need_update`

**Contract** — should the target be re-resolved this pass. The answer is read off the state
bit set.

```text
FUNCTION target_point_need_update() -> bool
  IF state HAS path_failed THEN RETURN true         # always re-resolve after a failure

  IF state IS exactly path_valid THEN
    IF NOT within distance_to_path_end of the end THEN
      IF target_actual AND NOT global_failed() THEN RETURN false
      IF last_time_target_set == 0 THEN RETURN true  # never resolved before
      RETURN (last_time_target_set + rebuild_time < now)   # throttle
    RETURN true                                      # logically arrived; re-resolve

  IF state HAS wait_new_path THEN RETURN false       # an answer is still coming
  IF state HAS no_path      THEN RETURN true
  IF state HAS path_end THEN
    # physically arrived: only worth re-resolving if we are not on the target vertex
    RETURN target_set.node IS NOT my current vertex

  RETURN false
```

**Notes** — the throttle is the reason creatures commit to routes. A valid path with a
current target is left alone until the rebuild interval elapses — five seconds by default —
which is why a creature pursuing a moving enemy visibly aims at where the enemy *was*.
Inside the global-failure window the actuality check is skipped, so a stuck creature
re-resolves every pass and keeps trying new random points.

"Logically arrived" (within the arrival tolerance along the path) and "physically arrived"
(the last waypoint passed) are handled separately and differently, which is worth keeping:
the first always re-resolves, the second only if the creature is not standing where it was
asked to stand.

## `find_target_point_set`

**Contract** — resolve the requested target. Two fast tests first; then, if neither
settles it, choose a position and find a vertex for it.

```text
FUNCTION find_target_point_set()
  target_found <- target_set

  IF target_type IS move_to THEN
    # fast test 1: the pair is self-consistent, accessible, and snaps onto the mesh
    IF valid_and_accessible(target_found) THEN RETURN

    # fast test 2: the position is forbidden. correct it to the nearest allowed one,
    # then immediately abandon it for a random point at scatter distance
    IF target_found.position is not accessible THEN
      snap target_found to the nearest accessible position and its vertex
      pick a random direction; candidate <- my position + direction * 30
      target_found <- set_target_accessible(candidate)
      IF target_found.node is known THEN RETURN

  target_found.node <- unknown

  # --- choose a position ---
  IF target_type IS retreat_from THEN
    direction <- normalize(my position - target_found.position)
    target_found.position <- my position + direction * 30

  IF target_found.position is not accessible THEN
    snap it to the nearest accessible position and its vertex

  # a target on top of the creature is no target at all: scatter it
  REPEAT up to 5 times
    IF target_found.position is within half a unit of my position THEN
      pick a random direction
      target_found <- set_target_accessible(my position + direction * 30)
    ELSE BREAK

  IF target_found.node is known THEN RETURN
  IF the position is not a valid mesh position THEN
    find_target_point_failed(); RETURN

  # --- a position without a vertex: go find one ---
  find_node()
```

**Notes** — fast test 2 is startling read plainly: on discovering the requested position is
forbidden, the element corrects it and then *throws the correction away* in favour of a
random point thirty units off in an arbitrary direction. This is the engine's answer to
"the state layer asked me to go somewhere I am not allowed" — not to approach the boundary
but to go somewhere else entirely. The visible result is a creature that, told to enter a
region it may not, wanders instead of pressing against the edge. Whether the correction
was meant to be used before being discarded is not recoverable.

A retreat becomes a destination thirty units directly away from the thing being fled. That
is the entire retreat implementation: no line of sight, no cover, no check that the
direction leads anywhere useful. If the resulting point is unreachable, the escalation
below handles it like any other.

The "within half a unit of myself" retry exists because the nearest-accessible correction
frequently returns the creature's own position — it is, after all, accessible. Five random
attempts at thirty units is the standard escape.

## `find_target_point_failed`

**Contract** — the give-up path, used inside the three-second global-failure window and
whenever a chosen position turns out not to lie on the mesh at all. Picks random points at
the scatter distance until one is both accessible and not the creature's own position,
giving up after five attempts and falling through to vertex resolution.

**Notes** — the loop's exit condition differs from the one above: here the loop breaks as
soon as a candidate is *far enough away*, regardless of whether a vertex was found, whereas
above it breaks as soon as one is *far enough away* and returns only if a vertex was found.
The difference is small and probably accidental.

## `find_node`

**Contract** — given a position with no vertex, find one, in four escalating stages. The
first that succeeds wins.

```text
FUNCTION find_node()
  # 1. is the position reachable in a straight line across the mesh from here?
  add a restriction border spanning me to the position
  node <- walk the mesh from my vertex toward the position
  remove the border
  IF node valid AND accessible THEN
    snap the position's height onto that cell; RETURN

  # 2. a direct lookup: which cell contains this position?
  IF the position is a valid mesh position THEN
    node <- the cell containing it
    IF node valid AND accessible THEN
      snap the position's height onto that cell; RETURN

  # 3. a cover point near the position, if cover approach is enabled
  IF use_covers THEN
    configure the cover evaluator with the position and the min/max/deviation band
    point <- best cover within `radius` of my position
    IF point EXISTS THEN
      target_found <- (point.position, point.vertex); RETURN

  # 4. the expensive fallback: the nearest mesh vertex within 30 units,
  #    found by running the chapter-14 search engine
  target_found.node     <- nearest vertex to the position within 30
  target_found.position <- that vertex's position
```

**Notes** — the ordering is a cost ladder and each rung answers a different question. Stage
one asks "can I walk straight there", which is both the cheapest and the most useful
answer because it also proves reachability. Stage two asks only "what cell is this in",
which is cheap but says nothing about getting there. Stage three substitutes a *tactically
chosen* nearby vertex — this is where "approach the enemy but end up behind something"
comes from, and it is enabled by the states that want it rather than by default. Stage four
runs a graph search and always produces something.

Stage four's thirty-unit range is a third occurrence of the same constant and, unlike the
other two, it is passed literally rather than through the named scatter distance. It is
almost certainly the same intent.

Note that stage three *replaces* the target rather than correcting it: the creature ends up
at a cover point near where it was told to go, not at the place itself. States that want to
arrive exactly where they asked must turn cover approach off.
