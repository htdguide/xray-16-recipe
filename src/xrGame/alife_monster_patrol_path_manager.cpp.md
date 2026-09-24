# src/xrGame/alife_monster_patrol_path_manager.cpp

> Walks an offline creature around an authored patrol path: picks where to join, decides which branch to take at each point, and decides what happens at a dead end.

**Needs** — [`alife_monster_patrol_path_manager.h`](alife_monster_patrol_path_manager.h.md) · [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/PatrolPath/patrol_path.h`](../xrAICore/Navigation/PatrolPath/patrol_path.h.md) · [`xrAICore/Navigation/PatrolPath/patrol_path_storage.h`](../xrAICore/Navigation/PatrolPath/patrol_path_storage.h.md) · [`xrAICore/Navigation/PatrolPath/patrol_point.h`](../xrAICore/Navigation/PatrolPath/patrol_point.h.md) · [`xrAICore/Navigation/game_graph.h`](../xrAICore/Navigation/game_graph.h.md)
**Used by** — reached through its declarations in [`alife_monster_patrol_path_manager.h`](alife_monster_patrol_path_manager.h.md); callers name that, not this file.
**Tier floor** — T2: graph walking over shared immutable level data

## Purpose

A **patrol path** is authored level data: a small graph of named points, each carrying a
game-graph vertex, a level-graph vertex and a precise position, joined by undirected
edges. This file is the cursor that moves an *offline* creature along one. It never moves
the creature itself — it only publishes "the point you should be heading for", which the
movement arbiter hands to the detail mover.

The file exists separately because its decisions are about the *path*, not about travel:
where to join a path that the creature may be standing in the middle of, which way to go
at a fork, and whether a dead end means stop or turn around. All three are authored
per-creature, not per-path, which is why they are settings on this object rather than
data on the path.

## State

```text
RECORD PatrolPathManager
  object                : ref MovementManagerHolder
  path                  : optional<ref PatrolPath>   # shared, immutable, owned by the level
  start_type            : FIRST | LAST | NEAREST | POINT | NEXT
  route_type            : STOP | CONTINUE
  use_randomness        : bool
  start_vertex_index    : int                        # only read when start_type is POINT

  actual                : bool     # the cursor below refers to the current path
  completed             : bool     # the route has ended and will not resume
  current_vertex_index  : int      # the point being travelled toward
  previous_vertex_index : int      # where we came from; excluded when choosing a branch
```

Invariants:

- `current_vertex_index` is a valid index into the path's points whenever `actual` holds.
- `previous_vertex_index` is the only mechanism preventing immediate backtracking; a
  freshly actualized cursor has none, so the first step at a fork may go anywhere.
- `completed` is only meaningful while `actual` — the reader combines them, so replacing
  the path resets "finished" implicitly rather than by an explicit clear.

The constructed defaults are the ones the shipped data relies on when a smart terrain
assigns a patrol without overriding them: join at the **nearest** point, **continue** at
a dead end (i.e. turn around and walk back), and **use randomness** at forks. A creature
with no path assigned is born `actual` and `completed`, which reads as "nothing to do"
and makes the update a single early return.

## `update`

**Contract** — one call per alife tick, from the movement arbiter. Cheap and allocation-
free in the common case; the expensive branch is `actualize`, which scans every point of
the path once.

```text
FUNCTION update()
  IF no path assigned          -> RETURN
  IF completed                 -> RETURN
  IF NOT actual                -> actualize()      # choose a joining point
  IF NOT location_reached()    -> RETURN           # still walking; nothing to decide
  navigate()                                       # arrived: pick the next point
```

**Invariants** — the ordering is the decision. Actualization happens *before* the
arrival test, so a path assigned this tick immediately produces a destination instead of
producing none for one tick. Arrival is tested *before* branching, so a creature that has
not arrived keeps the same destination and the destination is stable across ticks —
which matters because the detail mover caches a search against it.

## `actualize`

**Contract** — establishes a cursor on the current path and clears the finished flag.
Called only when the cursor does not refer to the current path.

```text
FUNCTION actualize()
  previous_vertex_index = none
  actual    = true
  completed = false
  SELECT ON start_type
    FIRST    -> current = 0
    LAST     -> current = point_count - 1
    NEAREST  -> select_nearest()
    POINT    -> current = start_vertex_index
    NEXT     -> FAIL WITH unsupported
```

**Notes** — `NEXT` ("resume from where the online creature left off") is declared in the
shared start-type enumeration but deliberately not handled here; the original calls it
far-fetched for an offline creature, which has no remembered online cursor to resume
from. A rebuild that wants it must first decide where the remembered cursor lives across
the online/offline transition, which is a state-ownership question, not a movement one.

## `select_nearest`

**Contract** — chooses the joining point for the `NEAREST` start type. Scans every point
of the path once. Never fails: a path always has at least one point.

```text
FUNCTION select_nearest()
  my_graph_vertex = object.game_vertex_id
  my_position     = game_graph.vertex(my_graph_vertex).game_point
  best            = none
  best_distance   = infinity
  FOR EACH point IN path.points
    IF point.game_vertex_id == my_graph_vertex
      best = point.index
      BREAK                      # exact match: stop, no better answer exists
    d = my_position.distance_to(game_graph.vertex(point.game_vertex_id).game_point)
    IF d < best_distance
      best_distance = d
      best = point.index
  current_vertex_index = best
```

**Invariants** — distance is measured **between game-graph vertices**, not between real
positions. That is the whole reason this is a separate routine from its online
counterpart: an offline creature does not have a real position, it has a graph vertex,
and the graph's own coordinates are the only comparable geometry. The consequence a
rebuild inherits: "nearest" is nearest *on the coarse graph*, so two patrol points inside
the same game-graph vertex are indistinguishable and the first one encountered wins.

The exact-match short circuit is not just an optimization — it makes the choice
deterministic when the creature stands on a graph vertex that the path visits, which is
the ordinary case for a smart terrain's own patrol.

## `location_reached`

**Contract** — true when the creature's recorded game-graph vertex *and* level-graph
vertex both equal the target point's. Both must match: the coarse vertex alone is too
loose (a game-graph vertex covers a large area), and on a level that is not loaded the
level vertex is carried as authored data and still compares equal, so the two-field test
works in both online and offline worlds without a special case.

## `navigate`

**Contract** — called only on arrival. Chooses the next point, or declares the route
finished. Consumes one draw from the creature's own random stream when randomness is
enabled — the creature's stream, not a global one, so that two creatures on the same
path diverge and a saved game reproduces the same choices.

```text
FUNCTION navigate()
  point = path.point(current_vertex_index)
  # candidate edges are all edges except the one we arrived along
  branching_factor = count of point.edges whose target != previous_vertex_index

  IF branching_factor == 0
    IF route_type == STOP
      completed = true
      RETURN
    ELSE IF route_type == CONTINUE
      IF point has no edges at all
        completed = true            # an isolated point: nowhere to continue to
        RETURN
      # a dead end with exactly one edge, the one we came in on: turn around
      SWAP current_vertex_index, previous_vertex_index
      RETURN

  chosen = use_randomness ? random_below(branching_factor) : 0
  next   = the chosen-th candidate edge's target
  previous_vertex_index = current_vertex_index
  current_vertex_index  = next
```

**Invariants** — excluding the arrival edge is what makes a patrol *sweep* rather than
oscillate: without it, a two-point path with randomness would half the time step straight
back. The dead-end rule then re-admits that edge explicitly, which is the only way a
linear path can be walked in both directions.

Choosing index zero when randomness is off makes an unbranched path deterministic and a
branched one take the author's first-declared edge every time — which is how a designer
pins a specific route without editing the path.

**Notes** — in the original, the three cases above do not actually return: each `break`
leaves the enclosing selection and falls into the branch-choosing code below with a
branching factor of zero, drawing a random number below zero and then scanning an edge
list that no longer corresponds to the (possibly swapped) cursor. A rebuild must return
at each of the three points, as written above; the intent is unambiguous and the
fall-through is a defect, not a behaviour the shipped data depends on.

## `path(name)` and the target accessors

**Contract** — `path(name)` resolves a patrol path by its authored name through the
level's patrol-path storage and installs it. `target_game_vertex_id`,
`target_level_vertex_id` and `target_position` read the three fields of the current
point; all three require a path and a valid cursor.

**Notes** — installing a path keeps the cursor valid *only if it is the same path*
(handled in the inline setter): assigning a different path invalidates the cursor so the
next update re-joins. Assigning the same path again is therefore free, which is what lets
a smart terrain re-issue a creature's job every tick without restarting its patrol.
