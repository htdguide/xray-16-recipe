# src/xrGame/alife_monster_detail_path_manager.cpp

> Walks an off-screen creature along the cross-level graph toward a destination, consuming game time at its travel speed and stepping it from vertex to vertex — the off-screen equivalent of walking.

**Needs** — [`alife_monster_detail_path_manager.h`](alife_monster_detail_path_manager.h.md) · [`alife_monster_brain.h`](../xrServerEntities/alife_monster_brain.h.md) · [`alife_smart_terrain_task.h`](alife_smart_terrain_task.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_time_manager.h`](alife_time_manager.h.md) · [`alife_graph_registry.h`](alife_graph_registry.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrServerEntities/xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`xrAICore/Navigation/game_graph.h`](../xrAICore/Navigation/game_graph.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/game_level_cross_table.h`](../xrAICore/Navigation/game_level_cross_table.h.md) · [`xrAICore/Navigation/graph_engine.h`](../xrAICore/Navigation/graph_engine.h.md)
**Used by** — reached through its declarations in [`alife_monster_detail_path_manager.h`](alife_monster_detail_path_manager.h.md); callers name that, not this file.
**Tier floor** — T3: a graph search and an arc-length walk.

## Purpose

An offline creature that has been told to go somewhere has to actually get there, over game
time, so that when the player arrives the world is arranged as if everyone had been walking
all along. This file is that motion: plan a route over the cross-level graph, then consume
it at the creature's travel speed each time the simulation updates.

The *detail* in the name is relative to the graph, not to the level: this is the fine
tracking of position along one graph edge, as opposed to the brain's choice of where to go.

## State

```text
RECORD DetailPathManager
  object             : reference to the creature's brain
  destination        : { graph_vertex, level_vertex, position }
  path               : list<graph vertex>   # reversed: the creature's current vertex
                                            #   is at the BACK, the destination at the
                                            #   front
  walked_distance    : real                 # metres covered along the current edge
  last_update_time   : int                  # game clock of the previous step
  speed              : real                 # metres per second at normal time scale

# Invariant: while a path is held, its back element equals the creature's current
#   graph vertex. Every step re-asserts it.
# Invariant: an empty path means the route search failed, which is the manager's only
#   failure state.
# Invariant: walked_distance is always less than the length of the edge from the back
#   of the path to the one before it.
```

**Notes** — The path is stored reversed so that advancing one vertex is a pop from the end
rather than an erase from the front. That is the whole reason the search's output is
reversed after it returns.

## `target`

**Contract** — Four forms, all setting the destination: a full triple of graph vertex, level
vertex and position; a graph vertex alone, which fills in that vertex's own representative
level vertex and point; a smart terrain task; and the same by reference.

**Invariants** — The full form asserts four things, and all four are conditional on the
destination being **on the currently loaded level** — off-level destinations carry only a
graph vertex, because no level vertex exists for them. On-level, the assertions require the
level vertex to be valid, the cross table to map it to the given graph vertex, and the
position to lie inside the level vertex. Together they state the invariant every position in
this engine satisfies: a position is a level vertex plus an offset, and the graph vertex is
derived from the level vertex, never chosen independently.

Setting a target does not invalidate the path. The next update notices the mismatch and
re-plans; see `actual`.

## `completed` / `actual` / `failed`

**Contract** — Three predicates that between them decide what the next update does.

```text
completed : the creature's graph vertex AND level vertex both equal the destination's
actual    : a path exists AND its front element is the destination's graph vertex
failed    : the path is empty
```

**Invariants** — *Actual* is what detects a changed target: the held path's front is the
destination it was planned for, so a new target makes the path stale without anyone having
to invalidate it. That is why `target` does not clear the path.

*Completed* compares both vertices, not the position: arriving means standing on the right
level vertex, and the fine position within it is set by the final step.

## `update`

**Contract** — Advances the creature by however much game time has passed since the last
step. Returns immediately if the clock has not advanced.

```text
FUNCTION update()
  now = game clock
  IF now <= last_update_time THEN RETURN
  update(now - last_update_time)
  last_update_time = game clock          # re-read, not reused

FUNCTION update(elapsed)
  IF this is the first ever step THEN RETURN      # the first delta is the whole game
  IF completed THEN RETURN
  IF NOT actual
    actualize()                                   # re-plan
    IF failed THEN RETURN
  follow_path(elapsed)
```

**Invariants** — The clock is read *again* after the work, not carried from before it. The
source says the difference is deliberately discarded: the route search can take real time
during which the game clock advances, and crediting the creature with that time would let a
re-planning creature jump forward. Discarding it costs a little travel and prevents the
jump.

The first update is skipped entirely because `last_update_time` starts at zero, so its delta
would be the whole elapsed game time and the creature would teleport to its destination.

## `actualize` — planning the route

**Contract** — Searches the cross-level graph from the creature's current vertex to the
destination's, constrained by the creature's terrain masks, and stores the result reversed.
On failure the path is left empty and the manager is in its failed state.

```text
FUNCTION actualize()
  path = empty
  search the game graph from the creature's vertex to the destination's,
    admitting only vertices whose terrain classification matches one of the
    creature's allowed terrain masks
  IF the search failed THEN RETURN            # path stays empty: failed state
  IF the path has one element
    REQUIRE it is the creature's current vertex     # already there
    RETURN
  walked_distance = 0
  reverse the path
  REQUIRE the path's back is the creature's current vertex
```

**Invariants** — The terrain mask constraint is what keeps a water creature out of the
buildings: the creature's section supplies a list of acceptable terrain classifications and
the search treats every other vertex as impassable. A rebuild needs the same filter as part
of the search's cost model, not as a post-filter — a post-filter would find no path where a
detour exists.

Resetting the walked distance is correct only because a re-plan starts from the creature's
current vertex, so any progress along the old edge is already accounted for by the creature
*being* at that vertex.

**Notes** — Failure is logged at length in checked builds: both endpoints with their level
names and their four terrain classification values, plus every mask the creature accepts.
That diagnostic is the only practical way to debug a level whose graph does not connect for
some creature kind, and it is worth reproducing.

## `follow_path` — consuming the route

**Contract** — Advances the creature by the elapsed game time, crossing as many graph
vertices as the time allows. Each crossing is a real registry move and a notification.

```text
FUNCTION follow_path(elapsed)
  REQUIRE not completed, not failed, path is actual
  IF the path's back is no longer the creature's vertex THEN make the path stale

  IF the path has one element                 # the destination vertex is reached
    snap the creature to the destination's level vertex and position
    walked_distance = 0
    RETURN

  remaining = elapsed in seconds
  WHILE the path has more than one element
    speed = on this level? the creature's on-level speed : its cross-level speed
    step  = remaining / world_time_scale * speed
    edge  = graph distance from the current vertex to the next one

    IF edge > step + walked_distance
      walked_distance = walked_distance + step
      RETURN                                  # still on this edge

    # The step carries past the next vertex: cross it, convert the leftover
    # distance back into time, and keep going.
    leftover = step + walked_distance - edge
    remaining = leftover * world_time_scale / speed
    walked_distance = 0
    pop the next vertex off the path
    move the creature to it in the graph registry
    REQUIRE the path's back is now the creature's vertex
    notify the creature that its location changed
```

**Invariants** — The travel speed is chosen *per edge*, not once: a creature on the loaded
level moves at its on-level speed and one elsewhere at its cross-level speed, and a route
crossing the boundary changes pace mid-walk. That is deliberate — the cross-level speed is a
world-map abstraction and the on-level one has to look right if the player watches the
creature arrive.

The leftover distance is converted back into *time* rather than carried as distance, because
the next edge may be walked at a different speed. Carrying distance would be wrong the
moment the creature crossed onto or off the loaded level.

The location-change notification fires per vertex crossed, not once per update, so a
creature that crosses three vertices in one step triggers three smart-terrain arrival
checks. That is required: skipping intermediate vertices would let a creature walk through a
smart terrain without it noticing.

Snapping to the exact destination position only happens on the final element, which is why
the destination's fine position is carried all the way through — the graph move alone would
leave the creature at the vertex's representative point.

## `on_switch_online` / `on_switch_offline`

**Contract** — Both clear the path. Crossing the online boundary invalidates the route in
either direction: going online, the creature's motion becomes the live pathfinder's problem;
going offline, its position was set by the live simulation and any held route is stale.

## `draw_level_position`

**Contract** — Where the world map should draw this creature: interpolated along the current
edge by the distance walked, when both endpoints are on the same level; otherwise its own
recorded position.

**Invariants** — The interpolation is refused across a level boundary because the two
vertices' representative points are in different levels' coordinate spaces and the line
between them is meaningless. The map then shows the creature parked at its last position
until it lands on the far side.
