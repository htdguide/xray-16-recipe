# src/xrGame/ai/monsters/control_path_builder.cpp

> The path resource: it is simultaneously the creature's movement manager and a control channel, so the whole navigation stack of chapter 14 is reachable as one bus resource.

**Needs** — [`control_path_builder.h`](control_path_builder.h.md) · [`control_manager.h`](control_manager.h.md) · [`movement_manager.h`](../../movement_manager.h.md) · [`detail_path_manager.h`](../../detail_path_manager.h.md) · [`level_path_manager.h`](../../level_path_manager.h.md) · [`level_location_selector.h`](../../level_location_selector.h.md) · [`game_location_selector.h`](../../game_location_selector.h.md) · [`xrAICore/Navigation/level_graph.h`](../../../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../../../xrAICore/Navigation/ai_object_location.h.md) · [`Actor.h`](../../Actor.h.md) · [`visual_memory_manager.h`](../../visual_memory_manager.h.md)
**Used by** — reached through its declarations in [`control_path_builder.h`](control_path_builder.h.md); callers name that, not this file.
**Tier floor** — T2: issues bounded path searches on the scheduled tick and answers geometry queries

## Purpose

This element is two things at once, and the doubling is the design. It is the creature's
**movement manager** — the chapter-23 object that owns the level path, the detailed path,
the location selectors and the restriction set — and it is also the **pure element on the
path channel**, whose payload any ability may seize and write. So an ability does not talk
to the navigation stack; it fills in a small record and this element applies it.

The payload is applied once per *scheduled* tick, not per frame, because applying it ends
in a path search.

## State

```text
RECORD PathPayload                       # what a capturer writes
  enable                  : bool         # may the creature move at all
  target_position         : vector
  target_node             : int          # a level-mesh vertex; all-ones means "unknown"
  path_type               : one of { level, game, patrol }
  game_graph_target_vertex: int          # only for the game path type
  use_dest_orientation    : bool
  dest_orientation        : vector       # the facing to arrive with
  try_min_time            : bool         # prefer the quickest route over the shortest
  extrapolate             : bool         # continue past the end rather than stopping dead
  velocity_mask           : int (bit set)# which gaits the planner may use
  desirable_mask          : int (bit set)# which gait it should prefer
  reset_actuality         : bool         # force a replan even if nothing moved
```

Plus everything the movement manager owns, which is chapter 23's.

## `update_schedule`

**Contract** — apply the payload and replan. Runs on the creature's scheduled tick. Refuses
to run at all — leaving the previous path standing — while the target position or target
node is outside the creature's current restrictions; then applies the payload's movement
parameters, issues the path update, and raises the path-updated event.

```text
FUNCTION update_schedule()
  # a restrictor change can leave the target momentarily inaccessible; wait it out
  IF path_type IS NOT patrol THEN
    IF target position not accessible
       OR (target node is known AND not accessible) THEN RETURN

  IF payload.reset_actuality THEN make_inactual()
  enable_movement(payload.enable)

  IF payload.enable THEN
    detailed path type <- smooth
    apply destination orientation, minimum-time preference, extrapolation
    apply the gait mask and the desirable gait
    SET the path type
    IF path type IS game THEN
      IF a game-graph target is set THEN
        aim at it and select by mask
      ELSE
        wander the game graph by random branching
    ELSE IF path type IS NOT patrol THEN
      IF the target node is unknown THEN RETURN    # nothing to plan toward
      set the detailed destination and the level destination vertex

  update_path()
  raise path_updated
```

**Notes** — the early return on an inaccessible target is the only place in the chapter
that *tolerates* a bad target rather than correcting it, and the comment in the source
gives the reason: a restrictor can change under the creature and both the target and the
creature's own node are briefly in an invalid state. Waiting is cheaper than correcting.

The unknown-target-node case returns silently. The source carries a note saying it should
be a hard failure naming the creature; it is not, and a creature whose ability forgot to
set a node simply stops moving with no diagnostic. A rebuild should make it loud.

Wandering the game graph by random branching, when no alife task has set a destination, is
how an off-task creature drifts across the world.

## `build_special`

**Contract** — replace the path with a straight line to a point, restricted to a gait mask,
and plan it **immediately** rather than on the next scheduled tick. Answers whether a
usable line was produced. Private; reached through the manager's access-checked wrapper.

```text
FUNCTION build_special(target, node, gait_mask) -> bool
  IF target not accessible THEN RETURN false
  IF node is unknown THEN
    # is the target reachable in a straight line from here?
    add a border to the restrictions spanning me to the target
    node <- walk the mesh from my vertex toward the target
    remove the border
    IF node invalid OR not accessible THEN RETURN false

  enable movement
  gait mask and desirable gait <- gait_mask
  no minimum-time preference, no destination orientation
  detailed path type <- smooth, path type <- level
  destination <- target, destination vertex <- node
  plan at once
  update_path()

  RETURN the path did not complete AND it was built this frame or later
```

**Notes** — every airborne and lunging ability in this chapter is built on this call: the
run-up before a jump, the run-out after landing, the skid of a rotation jump, the charge
of a run attack. Each computes a distance from its clip's duration and its gait's speed,
builds a line that far ahead, and locks the path channel so the ordinary planner will not
immediately replace it.

The temporary *border* added to the restrictions is what makes the straight-line walk
respect the creature's permitted region: without it the walk would cross a restrictor
boundary and the creature would run into a wall it is forbidden to pass.

The success condition is worth reading twice. A line is usable when the creature has *not*
already arrived and the detailed path's build timestamp is at or after the current frame —
that is, a fresh path exists. A stale path passing the first test but not the second is
reported as failure.

## `is_path_end`

**Contract** — is the creature within a given distance of the end of its path, measured
*along* the path rather than straight-line. A path that has not been built, or that the
creature is not moving on, or that is shorter than two waypoints, or whose current
waypoint is the last, all answer yes. Walks forward from the current waypoint summing
segment lengths and stops early once the budget is exceeded.

**Notes** — measuring along the path is what makes this usable for braking: the straight
line to the end of a path that doubles back would say "nearly there" while the creature
still has twenty units to walk.

## `is_moving_on_path`

**Contract** — the creature has a path it has not completed and movement is enabled. The
predicate everything in this chapter branches on to distinguish locomotion from standing.

## `valid_destination` / `valid_and_accessible` / `fix_position`

**Contract** — a three-step escalation used when a target has been chosen but not yet
trusted. `valid_destination` asks whether a (position, node) pair is self-consistent: both
are valid and the position lies inside the node's cell. `valid_and_accessible` adds the
restriction check and then *snaps the position's height* onto the cell's plane.
`fix_position` is that snap on its own.

**Notes** — the height snap is the important half. A target chosen by projecting a
direction, or handed in from script, has a plausible horizontal position and an arbitrary
height; leaving it produces a path that dives into or floats above the floor. The snap is
unconditional, so a caller must not use these on a target that is deliberately off the
mesh.

A debug build additionally checks that snapping did not move the position into a different
cell and logs both cells when it did. That check fires on genuinely mis-authored
restrictors and is the intended way to find them.

## `get_node_in_radius`

**Contract** — find an accessible mesh vertex in a random direction between two radii of a
source vertex, giving up after a fixed number of attempts. Answers whether one was found.
Each attempt adds a restriction border and walks the mesh toward the candidate, so the
answer is a vertex actually *reachable* in a straight line, not merely a nearby one.

**Notes** — this is the chapter's standard "go somewhere else" primitive: retreat, flee,
and the path builder's own failure recovery all use it. Random direction with a bounded
attempt count is a deliberate trade — it is cheap and it sometimes fails, and every caller
handles failure.

## `find_nearest_vertex`

**Contract** — the nearest mesh vertex to a position within a range, obtained by running
the chapter-14 search engine with the nearest-vertex policy. Exactly one vertex comes back
and an empty answer is a hard failure.

**Notes** — this is the *expensive* fallback. The path-builder base reaches it only after
the direct lookup and the cover search have both failed, precisely because it runs a graph
search to answer a geometric question.

## `on_travel_point_change` / `on_build_path`

**Contract** — the movement manager's hooks, re-raised as bus events so that any subscribed
ability learns that the creature passed a waypoint or that a path was built.

## `can_use_distributed_computations`

**Contract** — may this creature's path search be spread across frames. Answers no whenever
the player can currently see the creature, otherwise defers to the movement manager's own
rule.

**Notes** — this is a visible-quality decision, not an optimization: a creature the player
is watching must not visibly pause while its search completes over several frames, so it
pays for its search in one tick. Off-screen creatures take the cheap path. With no player
in the world at all — during load, or on a dedicated server — spreading is always allowed.

## `is_path_built`

**Contract** — every stage of the navigation stack is up to date. The predicate the
path-builder base uses to distinguish "no path yet" from "a path that leads nowhere".

## `reinit`

**Contract** — reset both halves — the movement manager and the channel element — and clear
the payload to: movement disabled, no destination orientation, minimum-time preferred, an
unknown target, all gaits admissible, and the level path type.

**Notes** — the source notes that reset runs twice, once from the creature and once from
the control manager, and flags it as something to fix. The operation is idempotent, so the
duplication is waste rather than a bug.

Minimum-time preference defaults to *on* here and to *off* in the base driver's own reset,
so the effective default depends on which reset ran last. Nothing records which was
intended.
