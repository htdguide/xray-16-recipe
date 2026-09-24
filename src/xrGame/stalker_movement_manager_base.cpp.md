# src/xrGame/stalker_movement_manager_base.cpp

> The bottom of a stalker's locomotion: choosing a destination it is allowed to stand at, a set of speeds it is allowed to use, and then reading back off the path what it is actually doing.

**Needs** — [`stalker_movement_manager_base.h`](stalker_movement_manager_base.h.md) · [`stalker_movement_manager_space.h`](stalker_movement_manager_space.h.md) · [`stalker_movement_params.h`](stalker_movement_params.h.md) · [`stalker_velocity_collection.h`](stalker_velocity_collection.h.md) · [`stalker_velocity_holder.h`](stalker_velocity_holder.h.md) · [`movement_manager.h`](movement_manager.h.md) · [`detail_path_manager.h`](detail_path_manager.h.md) · [`level_path_manager.h`](level_path_manager.h.md) · [`level_location_selector.h`](level_location_selector.h.md) · [`sight_manager.h`](sight_manager.h.md) · [`script_entity_action.h`](script_entity_action.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`stalker_movement_manager_base.h`](stalker_movement_manager_base.h.md)
**Tier floor** — T2: per-tick path and velocity work over a creature's navigation state.

## Purpose

Between the AI, which says "walk there, crouched, alarmed", and the pathfinder, which says
"here is a list of points and a speed at each", sits this. Its jobs, in the order the update
does them:

1. Make the destination *legal* — inside the creature's restrictors — before the pathfinder
   ever sees it.
2. Turn the four-part locomotion state into a bit mask of permitted velocities and hand that
   to the detail pathfinder, which then builds a path *whose points carry velocities*.
3. Read the current path point back and derive from it what the creature is actually doing —
   which may differ from what was asked for.

Step three is the one that surprises. The path is authoritative: the pathfinder may have
chosen to walk a corner the AI asked to run, and the creature's reported posture, gait and
mental state come back off the path point rather than out of the request.

## State

Two [movement params](stalker_movement_params.cpp.md) records, *current* and *target*, plus
a velocity table, the head rotation state, and a small memo for the is-something-in-my-way
query. See [`stalker_movement_manager_base.h`](stalker_movement_manager_base.h.md).

**Invariants**

- A creature is never both *at ease* and *crouched*. Asserted in the setters, re-asserted
  after the path is parsed, and re-asserted in the update. Three checks for one rule,
  because the pose it would produce does not exist.
- The current record is overwritten from the target at the top of every update, and then
  *mutated* by the path parsing. So "current" means "what the target became after the path
  had its say", not "what it was last frame".

## `reload`

**Contract** — read the creature's configuration section for the name of a speed table,
look that table up in the shared registry, and build the velocity masks from it.

**Notes** — speed tables are shared and named, so many creature types draw on a few tables.
The table is per *section*, which is the engine's general rule: the class supplies the
behaviour and the section supplies every number.

## `init_velocity_masks`

**Contract** — register, for each named locomotion combination, a linear speed and two
angular speeds: the one the pathfinder uses when *computing* whether a turn is affordable,
and the one the creature *actually* turns at.

```text
FOR EACH shipped combination
  register(mask, linear = table.speed(mental, posture, gait, direction),
                 compute_angular, real_angular = 2 x compute_angular)
```

**Invariants** — the standing-still combinations get zero linear speed and a full-turn-per-
second angular speed, which is what makes turning on the spot fast. Every moving combination
gets its real turn rate at twice its planning turn rate, so the pathfinder is *pessimistic*
about corners: it plans as if the creature turns half as fast as it does, and the creature
then makes every corner it planned with margin. A rebuild that plans with the true rate will
produce paths the creature physically cannot follow and a visible wobble at every corner.

**Notes** — the planning angular speeds differ by mental state in a way that encodes the
whole feel of the movement. Relaxed walking plans at about a hundred degrees a second;
relaxed running plans at about eleven, which is almost straight — a relaxed running stalker
commits to long straight runs and takes wide corners. Every *alarmed* combination plans at a
hundred times a half-turn, effectively unbounded, which means an alarmed creature will accept
any corner at all. Panic running plans like relaxed running: a fleeing creature also runs
straight.

That single table is why a stalker walking home rounds corners smoothly and a stalker under
fire jinks.

The backward-moving variants are registered with the *forward* speeds from the table. A
stalker retreating backwards moves as fast as it advances. Whether that is a decision or an
oversight is not recoverable from the source; it is audible in play as creatures backing away
implausibly quickly.

## `reinit`

**Contract** — reset both movement records to defaults, set the body turn rate to a full turn
per second and the head turn rate to three quarters of that, and forget the last turn point.

**Notes** — the head turns *slower* than the body by default when alarmed, and much slower —
a quarter turn per second — when relaxed and the sight system is in charge. That difference
is what makes a relaxed stalker's head drift and an alarmed one's head snap.

## `initialize`

**Contract** — put the creature into the default locomotion state: level path, smooth detail
path, standing, not moving, alarmed, no forced facing. Then **remove every movement
restrictor** and place the creature at the nearest position it may legally occupy.

**Invariants** — clearing the restrictors before finding a legal position is deliberate: at
initialization the creature may be standing somewhere its eventual restrictors forbid, and
the only way out of that is to start from an unrestricted world.

## `setup_movement_params`

**Contract** — hand the pathfinder a destination, guaranteeing it is one the creature's
restrictors allow. Never leaves an illegal destination in place.

```text
FUNCTION setup_movement_params(params)
  pathfinder.path_type := params.path_type
  IF the path type is a game path or a patrol path THEN
    clear the exact destination          # those path types carry their own
  detail_pathfinder.type := params.detail_path_type
  level_pathfinder.evaluator := the base cost model

  IF params has an exact destination THEN
    IF that position is forbidden THEN
      substitute the nearest allowed position, and its vertex
    detail_pathfinder.destination := that position
  ELSE IF the path type plans through the navigation mesh THEN
    IF the destination vertex is forbidden THEN
      substitute the nearest allowed vertex and its position
    detail_pathfinder.destination := the vertex's position

  IF params has a forced facing THEN
    detail_pathfinder.destination_orientation := that direction, enabled
  ELSE
    disable destination orientation
```

**Invariants** — **the pathfinder is never given an illegal destination.** Every branch
either verifies legality or substitutes the nearest legal alternative. That is the whole
purpose of this function; the pathfinder itself has no restrictor model, and a forbidden
destination there produces a failed path rather than a corrected one.

Game paths and patrol paths *clear* the exact destination, because those path types are
sequences authored in the level data and an exact position would fight the sequence.

## `setup_velocities`

**Contract** — compose the four-part locomotion state into a bit mask, and give the detail
pathfinder two masks: the velocities it should *prefer* and the velocities it may *use*.

```text
FUNCTION setup_velocities(params)
  mask := forward-motion flag
  mask |= crouch or stand, per posture
  mask |= free, danger or panic, per mental state
  CASE gait
    walk : mask |= walk
    run  : mask |= run
    else : mask |= standing;  clear both direction flags

  IF the creature is alarmed THEN
    preferred := mask
    allowed   := mask + standing
  ELSE
    ask the pathfinder to minimize time rather than distance
    preferred := mask + standing
    allowed   := mask + walk + standing
```

**Invariants** — the two masks are not the same, and the difference is the decision. An
**alarmed** creature may only use the gait it was told to use, or stand: it will not
silently walk a run. A **relaxed** creature is allowed to walk even when asked to run, and
the pathfinder is switched to minimizing time, so it picks whichever of the two gets there
sooner given the corners.

That is why relaxed stalkers slow to a walk around obstacles and alarmed ones do not. It is
also why an alarmed creature's path can simply fail where a relaxed one's succeeds.

Clearing the direction flags in the standing case is required: a standing velocity with a
direction flag has no table entry.

## `parse_velocity_mask`

**Contract** — the reverse direction. Read the velocity at the current path point, set the
creature's linear and angular speed from it, and **overwrite the current movement record's
posture, gait and mental state with what the path point says**. Also decides, at each turn,
whether the sight system or the movement layer owns the head.

```text
FUNCTION parse_velocity_mask(params)
  IF the path point changed since the last turn THEN forget the last turn point
  sight_owner := the sight system (restored on exit unless changed below)

  IF not moving, or there is no usable path, or the path is done or stale THEN
    speed := 0
    body turn rate := quarter-turn per second if relaxed, full turn if alarmed
    params.gait := stand still
    set the head speed from the mental state
    RETURN

  point    := the current path point
  velocity := the velocity registered for that point

  IF that velocity is zero — a turn-in-place point THEN
    IF relaxed THEN
      aim the body and head down the next path segment
      take the head away from the sight system        # the turn owns it
    IF alarmed, OR the turn is already aligned, OR we already turned here THEN
      remember this point as turned
      give the head back to the sight system
      advance to the next path point and use its velocity instead
  ELSE IF relaxed or panicking THEN
    IF panicking AND the next segment turns by more than about 11 degrees THEN
      re-read the velocity as its ALARMED variant         # see Notes
    ELSE IF relaxed AND the turn exceeds what this velocity can turn at THEN
      aim the body down the segment, take the head from the sight system,
      and replace the velocity with: zero linear, half-turn-per-second angular

  speed          := velocity.linear
  body turn rate := velocity.real_angular
  params.posture, params.mental_state, params.gait := decoded from the point's velocity mask
  set the head speed from the mental state
```

**Invariants** — after this function, the current record describes what the creature is
*doing*, which may differ from what was asked. A caller that needs to know what it asked for
must read the target record.

The sight-ownership handover is scoped: whatever it is set to, it is applied once on exit.
That matters because the function has many early returns, and a head left detached from the
sight system stays detached forever.

**Notes** — the panic rule is the most interesting line in the file. A panicking creature
that meets a corner sharper than about eleven degrees has its velocity *rewritten as the
alarmed variant* for that point. Panic speeds are tuned for straight-line flight and cannot
turn; alarmed speeds can. So a fleeing stalker automatically drops out of full flight for
each corner and back into it on the straight. There is no state for this and no transition:
it is one mask substitution per path point.

The relaxed rule is the mirror: a relaxed creature that meets a corner it cannot turn at
simply *stops and turns*, then resumes. That is the origin of the characteristic pause-and-
pivot of stalkers walking around the world, and it is why the relaxed angular speeds in the
velocity table are effectively a "how sharp a corner will I take without stopping" setting.

The turn-in-place branch's bookkeeping — remembering which path point a turn was performed
at — exists to stop the creature turning twice at the same point, which would deadlock: the
turn completes, the alignment test passes, the point is advanced. Without the memo a
creature at an awkward corner oscillates.

In diagnostic builds, a relaxed mental state decoded off a path point while the creature is
in combat and not executing a wounded enemy is reported as a fault. That is a real invariant
of the AI above — nothing in combat should be relaxed — checked at the one place it is
observable.

## `check_for_bad_path`

**Contract** — downgrade a run to a walk when the path ahead turns too sharply within the
next couple of metres. Applies only to an alarmed creature that is running and has not
arrived.

```text
FUNCTION check_for_bad_path(params)
  IF not running, or not alarmed, or already arrived THEN RETURN
  IF fewer than two path points remain THEN RETURN
  direction := the current segment's direction
  distance  := how far to the next point
  FOR EACH following segment
    distance += its length
    IF the angle between it and the current segment > 67.5 degrees THEN
      params.gait := walk
      RETURN
    IF distance >= 2 units THEN RETURN
```

**Invariants** — the lookahead is bounded by *distance*, not by point count: two units
ahead, whatever the point spacing. That is what makes the rule independent of how finely the
pathfinder happened to subdivide the route.

**Notes** — this is the only place the layer overrules the AI's gait outright, and it is
narrow on purpose: only alarmed running, only a turn sharper than three-quarters of a right
angle, only within two metres. It exists because an alarmed creature's velocity mask forbids
walking (see `setup_velocities`), so without this the creature would attempt the corner at a
run and overshoot visibly. The two rules are a pair and a rebuild must carry both or neither.

## `update`

**Contract** — one tick. Copies target into current, then runs the pipeline. Does nothing
when the manager is disabled.

```text
FUNCTION update(time_delta)
  IF disabled THEN RETURN
  current := target
  IF forced, or we are meant to be moving THEN
    setup_movement_params(current)
  IF a script is driving this creature's speed THEN RETURN      # see Notes
  IF forced, or we are meant to be moving THEN
    setup_velocities(current)
    update_path()
  parse_velocity_mask(current)
  check_for_bad_path(current)
```

**Invariants** — the three "meant to be moving" guards are one optimization with a visible
consequence: a creature standing still does **not** re-plan, so its path stays whatever it
was. The forced-update flag exists to defeat exactly that, and is used by the action that
shoots past the player (see
[`stalker_combat_actions.cpp`](stalker_combat_actions.cpp.md)), where the latency is
dangerous.

Script control returns *early*, after the destination has been set up but before velocities
and path. A script that has set an explicit speed owns the locomotion, and the remaining
stages would overwrite it.

## `set_nearest_accessible_position`

**Contract** — place the creature's destination at the nearest position it is allowed to
occupy, given a wanted position and vertex. Two-argument form takes them; no-argument form
uses the creature's own current position and vertex.

```text
FUNCTION set_nearest_accessible_position(position, vertex)
  IF position is not actually inside vertex THEN
    position := the vertex's own position
  ELSE
    snap position's height to the vertex's ground plane at that x,z

  IF the position is forbidden THEN
    vertex, position := the nearest allowed pair
  ELSE IF the vertex is forbidden THEN
    vertex, position := the nearest allowed pair, searched from the vertex's position

  set the destination vertex and the exact destination
```

**Invariants** — position and vertex must agree: the position must lie inside the vertex's
cell and on its ground plane. The height snap is what enforces the second half, and it
matters because a position half a metre above the floor makes the creature path to a place
it then falls from.

Both the position and the vertex are checked for legality independently, because a legal
position can lie in an illegal vertex and the reverse.

## `on_restrictions_change`

**Contract** — when the creature's restrictor set changes, if its current destination has
become illegal, relocate to the nearest legal position.

**Invariants** — this is the mechanism that makes the anomaly escape work (see
[`stalker_anomaly_actions.cpp`](stalker_anomaly_actions.cpp.md)): the action forbids the
zones the creature is standing in, and the notification does the relocating. Without it,
adding a restrictor would leave the creature walking to a destination it may no longer
occupy.

## `is_object_on_the_way` and `update_object_on_the_way`

**Contract** — does the given object sit on or near this creature's path within the next
`distance` units. Memoized: the answer is recomputed only when the object, the distance, the
creature's position or the object's position has changed since the last query.

```text
FUNCTION recompute(object, distance)
  IF the path is stale, empty, or we are on its last point THEN RETURN
  travelled := 0
  FOR EACH remaining path segment
    IF the object is within 1 unit of that segment THEN answer := true;  RETURN
    travelled += segment length
    IF travelled > distance THEN RETURN          # far enough: no
```

**Invariants** — the memo is invalidated wholesale whenever a new path is built. A memo
keyed on positions alone would survive a re-route and answer about a path that no longer
exists.

**Notes** — the geometry is point-to-segment distance with both endpoint cases handled: an
object behind the segment's start or beyond its end is measured to that endpoint rather than
to the infinite line. Getting that wrong makes objects "on the way" when they are beside the
corner the creature already turned.

One unit is the corridor half-width — roughly a body's width. The query is what the obstacle
layer above uses to decide whether another creature is blocking, and what the AI uses to
decide whether the player is in the line of fire.

## `speed(direction)`

**Contract** — the tuned linear speed for the creature's *current* posture, gait and mental
state in the given direction. Asserts that the creature is not standing still, because there
is no table entry for that.

## `setup_speed_from_animation`

**Contract** — override the creature's speed with one derived from the animation currently
playing. This is the channel by which a root-moving animation drives locomotion instead of
the other way round.

## `set_level_dest_vertex`

**Contract** — set the pathfinding destination vertex, and **clear the target smart-cover
identity**.

**Invariants** — the clearing is the load-bearing half. A smart cover is addressed by
identity rather than by position (see
[`stalker_combat_action_base.cpp`](stalker_combat_action_base.cpp.md)); setting an ordinary
destination while a smart cover is still named would leave the two fighting, with the
smart-cover layer trying to enter a cover the creature is walking away from.

## `on_build_path` and `remove_links`

**Contract** — clear the object-on-the-way memo. The first on every new path, the second
when an object is being destroyed and every reference to it must be dropped.
