# src/xrGame/stalker_movement_manager_smart_cover_loopholes.cpp

> Routes a stalker through a smart cover — picks which aperture to enter by,
> which sequence of transitions to play, and which one to leave by.

**Needs** — [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`smart_cover.h`](smart_cover.h.md) · [`smart_cover_loophole.h`](smart_cover_loophole.h.md) · [`smart_cover_transition.hpp`](smart_cover_transition.hpp.md) · [`smart_cover_transition_animation.hpp`](smart_cover_transition_animation.hpp.md) · [`smart_cover_animation_selector.h`](smart_cover_animation_selector.h.md) · [`cover_manager.h`](cover_manager.h.md) · [`sight_manager.h`](sight_manager.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`ai_space.h`](ai_space.h.md) · [`Level.h`](Level.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: graph search over a handful of nodes, plus one ray query per candidate.

## Purpose

A **smart cover** is an authored object whose description is a small directed graph: the
nodes are **loopholes** (named apertures, each with a position, a direction, a
field-of-view cone, a range and flags saying whether a creature may enter or leave the
cover through it), and the edges are **transitions**, each carrying a list of candidate
**actions**, each action carrying one or more animations that move a creature from one
loophole to the next. Two pseudo-loopholes, written here as `enter` and `exit`, stand for
"the world outside", so that entering and leaving are edges of the same graph as moving
between apertures.

This file is everything that decides a *route* through that graph, and the one thing that
decides *which animation variant* of a chosen edge to play. It does not play animations
and it does not drive the body; it produces a loophole path and the transition sitting at
its head, which the rest of the movement manager consumes.

Note on the pseudo-loopholes: the encoding is "the empty loophole name, tagged as an
entry or as an exit". The important property is that the two directions are distinct
tokens, so that a path may legitimately start with `enter` and end with `exit`.

## State

Stateless in itself; it reads and writes the movement layer's
`path`, `temp_path`, `current_transition` and `current_transition_animation`
(see [the header](stalker_movement_manager_smart_cover.h.md)), and it reads the `current`
and `target` movement-parameter records.

## `enter_path`

**Contract** — given a world position, a navigation vertex and a target cover, finds the
cheapest way *into* that cover ending at a named loophole, and optionally writes the
resulting loophole sequence. Returns the cost. Fails if the target loophole does not exist
in the cover. Allocates only into the supplied path buffer.

The cost of entering by a given enterable loophole is the sum of two things measured in
different units, which is the decision worth recording: the straight-line distance from
the creature to that loophole's firing position, plus the graph cost of the route from
there to the target loophole, which comes out of the shortest-path search the cover's
transition graph is walked with. Walking distance and animation cost are added as if
comparable; the authored edge weights are what makes that reasonable.

```text
FUNCTION enter_path(out_path, position, level_vertex, cover, target_loophole_id) -> real
  best <- +infinity
  FOR EACH loophole IN cover.description.loopholes
    IF NOT loophole.enterable
      CONTINUE
    loophole_path(cover, loophole.id, target_loophole_id, temp_path)
    first <- temp_path.front()
    cost  <- distance(position, cover.fov_position(first))
           + shortest_path_cost_of_the_search_just_run
    IF cost >= best
      CONTINUE
    best <- cost
    IF out_path EXISTS
      out_path <- temp_path          # swap, not copy: the buffer is reused each frame
  IF out_path EXISTS
    out_path.push_front("enter")     # the route begins outside the cover
  RETURN best
```

**Notes** — reading the search cost back out of the path-finding engine after the fact,
rather than having `loophole_path` return it, is an artifact of a shared global search
engine. It makes the two calls order-dependent: the cost read must immediately follow its
search. A rebuild should return the cost from the search.

## `build_enter_path`

**Contract** — computes the route from outside into the *target* cover's target loophole
and installs it, together with the first transition and that transition's animation. If
the route has only one element there is nothing to traverse and the current transition is
cleared.

```text
FUNCTION build_enter_path()
  target_loophole <- as_exit_token(target_params.cover_loophole_id)
  clear path
  enter_path(path, self.position, self.level_vertex, target_params.cover, target_loophole)
  install_head_transition()
```

where `install_head_transition` is the step repeated by every path builder here:

```text
FUNCTION install_head_transition()
  IF path.size > 1
    current_transition <- action(cover, path[0], path[1])
    current_transition_animation <- current_transition.animation
  ELSE
    current_transition <- none
    current_transition_animation <- none
```

## `build_exit_path`

**Contract** — computes the route from the current loophole out of the cover toward the
creature's desired destination in the world, and installs it. Considers every *exitable*
loophole and scores each by three added terms.

```text
FUNCTION build_exit_path()
  best <- +infinity
  FOR EACH loophole IN current_cover.description.loopholes
    IF NOT loophole.exitable
      CONTINUE
    loophole_path(current_cover, current_loophole.id, loophole.id, temp_path)
    cost <- shortest_path_cost_of_the_search_just_run          # moving inside the cover
          + weight_of_edge(loophole.id, "exit")                 # playing the exit animation
    target_position <- target_params.desired_position
                       OR position_of(level_dest_vertex)
    (exit_position, exit_vertex) <- nearest_action(current_cover, loophole.id, "exit",
                                                   target_position,
                                                   target_body_state = target.body_state)
    cost <- cost + exit_path_weight(exit_vertex, exit_position,
                                    level_dest_vertex, target_position)
    IF cost < best
      best <- cost
      path <- temp_path
  path.push_back("exit")
  install_head_transition()
```

**Invariants** — at least one loophole must be exitable, or the creature is trapped; the
code asserts rather than handling it, which is the right call: a cover with no exit is
broken authored data and should fail at load.

## `exit_path_weight`

**Contract** — the cost of walking from where the exit animation deposits the creature to
where it actually wants to be. It is the straight-line distance, ignoring the navigation
vertices it is handed. That is a deliberate approximation: a real path search here would
be run once per exitable loophole per re-plan, and the exits of one cover are close enough
together that the ranking rarely differs.

## `test_pick`

**Contract** — true when nothing opaque stands between two positions, used to reject a
transition whose animation would deposit the creature through a wall. Traces against the
static collision database only, from two metres above each end point rather than from the
points themselves, because the points are on the floor and a floor-level ray clips scenery
it should not. A surface counts as blocking when the creature's own vision system rates it
less than fully transparent — the same material transparency the senses use, so that
"cannot walk there" and "cannot see there" agree.

```text
FUNCTION test_pick(source, destination) -> bool
  source.height      <- source.height + 2
  destination.height <- destination.height + 2
  direction <- destination - source
  distance  <- length(direction)
  IF distance is negligible
    RETURN true                     # the two points coincide; nothing to block
  ray_query(static geometry only, source, normalize(direction), distance) WITH
    ON each hit:
      IF vision_material_transparency(hit.object, hit.element) < 1
        record hit.range; STOP
  RETURN no hit was recorded
```

## `nearest_action`

**Contract** — chooses which of an edge's candidate actions, and within it which
animation variant, a creature should play to get from one loophole to another given where
it wants to end up. Returns the action and, through its outputs, the world position the
animation starts from and the navigation vertex there. Fails loudly when no candidate
survives, because that means the authored transition cannot be performed at all.

A candidate animation is rejected when any of these fails, in this order — the order is
cheapest-test-first and is worth preserving:

1. its start position is farther from the requested position than the best found so far;
2. if it has an animation at all, its start position must lie on the navigation mesh, must
   resolve to a valid vertex, and the mesh height at that vertex must agree with the
   animation's own height to within two metres — a mismatch means the animation would
   start in the air or in the floor;
3. and the creature must be able to reach it without passing through geometry
   (`test_pick`).

Body state acts as a soft preference rather than a filter: a candidate with the wrong
posture is kept if nothing with the right posture has been found yet, and dropped as soon
as one has. This is what lets a crouching stalker still exit a cover that only offers a
standing exit animation.

```text
FUNCTION nearest_action(cover, from_loophole, to_loophole, position,
                        OUT start_position, OUT start_vertex, target_body_state)
       -> TransitionAction
  edge <- cover.description.transitions.edge(from_loophole, to_loophole)
  best <- none ; best_distance <- +infinity ; best_body_state <- unset
  FOR EACH action IN edge.actions
    IF NOT action.applicable      # preconditions authored on the action
      CONTINUE
    FOR EACH animation IN action.animations
      candidate <- cover.transform applied to animation.position
      d <- squared_distance(candidate, position)
      IF d > best_distance
        CONTINUE
      vertex <- unset
      IF animation.has_animation
        IF candidate is off the navigation mesh                    -> CONTINUE
        vertex <- vertex_at(candidate)
        IF vertex invalid                                          -> CONTINUE
        IF mesh_height(vertex, candidate) differs from candidate.height by > 2
                                                                   -> CONTINUE
        IF NOT test_pick(self.position, candidate)                 -> CONTINUE
      IF target_body_state EXISTS
         AND animation.body_state <> target_body_state
         AND best_body_state == target_body_state
        CONTINUE                  # we already have a posture-correct candidate
      best_body_state <- animation.body_state
      best_distance <- d ; best <- action
      start_position <- candidate ; start_vertex <- vertex
  RETURN best                      # absence is a fatal data error
```

## `action`

**Contract** — the cheap sibling of `nearest_action`: the first *applicable* candidate
action on an edge, with no geometric selection at all. Used wherever the path is being
installed rather than evaluated, since by then the edge is already committed and only the
animation identity is needed.

## `build_exit_path_to_cover`

**Contract** — the route for moving from one smart cover directly into another, which is
neither an exit nor an entry but both. Scored per exitable loophole of the current cover
as: the in-cover route cost, plus the exit edge's weight, plus the cost of entering the
target cover from wherever that exit deposits the creature.

The target loophole of the destination cover is used if it is enterable; if it is not, the
nearest enterable loophole is substituted. The installed path covers only the *current*
cover's portion plus the `exit` token; the entry into the next cover is re-planned when
the creature is actually outside, because by then its real position is known.

```text
FUNCTION build_exit_path_to_cover()
  target_position <- target_cover.transform applied to target_loophole.fov_position
  best <- +infinity
  FOR EACH loophole IN current_cover.description.loopholes WHERE loophole.exitable
    loophole_path(current_cover, current_loophole.id, loophole.id, temp_path)
    cost <- shortest_path_cost_of_the_search_just_run
          + weight_of_edge(loophole.id, "exit")
    (exit_position, exit_vertex, action) <-
        nearest_action(current_cover, loophole.id, "exit", target_position,
                       target_body_state = standing)
    cost <- cost + enter_path(none, exit_position, exit_vertex, target_cover,
                              (target_loophole.enterable ? target_loophole
                                                         : nearest_enterable_loophole).id)
    IF cost < best
      best <- cost ; selected_action <- action ; path <- copy of temp_path
  path.push_back("exit")
  IF path.size > 1
    current_transition <- selected_action          # NOT re-derived from the path head
    current_transition_animation <- current_transition.animation
  ELSE
    clear both
```

**Notes** — the copy of `temp_path` here (rather than the swap the other builders use) is
forced: `enter_path` is called inside the loop and reuses `temp_path` for its own
candidates, so the candidate route must be preserved across that call. This is the one
place where the shared scratch buffer costs a copy, and it is a good argument for giving
the two searches separate buffers in a rebuild.

Note also that this builder installs the action chosen by `nearest_action` rather than the
first applicable action on the head edge. That is the correct behaviour — the geometric
choice is the whole point — and it means the two path builders disagree about which
transition variant gets installed. Preserve the difference.

## `actualize_path`

**Contract** — dispatches to the right path builder from the pair (current cover, target
cover). Requires that at least one of the two exists.

```text
FUNCTION actualize_path()
  IF no current cover              -> build_enter_path()        # we are outside, going in
  ELSE IF no target cover          -> build_exit_path()         # we are inside, going out
  ELSE IF current cover <> target  -> build_exit_path_to_cover()
  ELSE                                                          # same cover, new loophole
    loophole_path(current_cover,
                  as_enter_token(current_loophole_id),
                  target_params.cover_loophole_id,
                  path)
    install_head_transition()
```

## `try_actualize_path`

**Contract** — rebuilds the path only when it has gone stale, which is the thing that
makes this system affordable: the route is recomputed on a change of endpoint, not every
frame. Stale means any of: the path is empty; its head no longer names where the creature
is; or its tail no longer names where the creature is going.

```text
FUNCTION try_actualize_path()
  IF path is empty                                     -> actualize_path(); RETURN
  IF path.front() <> as_enter_token(current_loophole)  -> actualize_path(); RETURN
  IF path.back()  == as_exit_token(target_loophole_if_same_cover)  -> RETURN
  actualize_path()
```

## `nearest_enterable_loophole`

**Contract** — the loophole a creature standing outside would actually enter the target
cover by: the second element of the freshly actualized enter path, the first being the
`enter` token. Only meaningful when the creature is outside and a target cover is set.

## `next_loophole_id`

**Contract** — the loophole the creature moves to next, after refreshing the path. Only
meaningful inside a cover.

## `go_next_loophole`

**Contract** — advances the creature one node along the path. This is the state machine
of cover occupancy and its three cases are the whole of it.

```text
FUNCTION go_next_loophole()
  try_actualize_path()
  IF path.size == 1
    RETURN                          # already where we want to be
  IF path[0] is the "enter" token
    current.cover_id       <- target.cover_id        # we are now inside
    current.cover_loophole <- path[1]
    RETURN
  IF path[1] is the "exit" token
    current.cover_id <- ""          # we are now outside
    on_smart_cover_exit()
    RETURN
  current.cover_loophole <- path[1]
```

**Invariants** — when the head is `enter` there must be a target cover and no current
cover; when the next node is `exit` the path must have exactly two elements, since exit
is terminal.

## `non_animated_change_loophole`

**Contract** — the walked alternative to animated loophole changes, used when a script
turns animation off. The creature is driven to the next loophole's firing position as if
it were any other destination; only when both the path and the aim have converged does it
actually occupy the loophole and restart the in-cover animation planner.

The sight rule here is the reason `apply_loophole_direction_distance` exists: far from the
loophole the creature looks where it is walking; within that distance it looks along the
loophole's authored enter direction, so it arrives already facing out of the aperture
rather than snapping round on arrival.

```text
FUNCTION non_animated_change_loophole()
  setup_movement_params()
  IF NOT non_animated_loophole_change
    RETURN
  run the layer below with the current movement parameters
  next <- next_loophole_id()
  IF next is not the "exit" token
    IF NOT target_approached(apply_loophole_direction_distance)
      sight <- follow the path direction
    ELSE
      sight <- look along cover.enter_direction(loophole(next))
  IF path not completed              -> RETURN
  IF sight target not reached        -> RETURN
  go_next_loophole()
  IF no current cover                -> RETURN     # we just walked out
  animation_selector.on_animation_end()
  animation_selector.planner.update()
```

## `setup_movement_params`

**Contract** — translates the next node of the path into movement parameters for the layer
below: a destination navigation vertex, a desired position and a body state. Always sets
the movement type to running. Clears the non-animated-change flag when there is nowhere
left to go.

```text
FUNCTION setup_movement_params()
  try_actualize_path()
  IF path.size == 1
    non_animated_loophole_change <- false ; RETURN
  next <- path[1]
  current.movement_type <- run
  IF next is the "exit" token
    RETURN                          # the exit animation owns the motion from here
  current.body_state <- current_transition_animation.body_state
  vertex   <- cover.level_vertex_id(loophole(next))
  position <- cover.fov_position(loophole(next))
  # both must lie inside this creature's restrictors, or the destination is unreachable
  set level destination vertex to `vertex`
  current.desired_position <- position
```

## `loophole`

**Contract** — resolves a loophole by identifier within a cover's description. Fails with
the cover and loophole names when absent, because the names come from authored data. The
comparison is on interned-string identity rather than character content, which is what
makes this linear scan cheap enough to sit in a per-frame path.

## `idle_min_time` / `idle_max_time` / `lookout_min_time` / `lookout_max_time`

**Contract** — four get/set pairs forwarded verbatim to the in-cover animation planner.
They bound how long a stalker dwells in an idle pose and how long it exposes itself in a
lookout pose, and scripts set them per encounter. No logic here.

## `start_non_animated_loophole_change` / `stop_non_animated_loophole_change`

**Contract** — enter and leave the walked-loophole-change mode. Starting it unbinds the
global animation selector (so the creature stops being driven by cover animations),
raises the flag and immediately performs one change step; stopping it lowers the flag and
rebinds the selector. The order matters in both directions: unbind before the flag on,
flag off before rebind, so that no frame observes an animation selector driving a creature
that is also walking.

## `position_to_cover_from`

**Contract** — the position a cover is being taken *against*: the thing the creature wants
the cover to protect it from. Resolved in priority order, and the priority is a design
decision rather than a fallback chain.

```text
FUNCTION position_to_cover_from() -> Position
  IF target.cover_fire_position EXISTS                # a script named a place
    RETURN it
  IF target.cover_fire_object EXISTS                  # a script named an object
    IF self is dead            -> RETURN the object's true position
    info <- memory.of(object)
    IF info has any of visual, sound or hit evidence
      RETURN info.object_params.position              # what we believe, not what is
    RETURN the object's true position                 # we were told about it out of band
  enemy <- memory.enemy.selected
  IF enemy IS none
    RETURN self.position                              # cover against nothing: stay put
  RETURN memory.of(enemy).object_params.position
```

**Notes** — the dead-creature branch is not decoration. A corpse's movement manager is
still queried while its ragdoll settles, and its memory is no longer being updated, so the
only sane answer is the object's real position.
