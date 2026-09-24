# src/xrGame/patrol_path_manager.cpp

> Walks a creature along an authored waypoint graph: choose where to join it, then at each waypoint pick an outgoing edge by weighted chance, skipping anything the creature is not allowed to reach.

**Needs** — [`patrol_path_manager.h`](patrol_path_manager.h.md) · [`patrol_path_manager_inline.h`](patrol_path_manager_inline.h.md) · [`restricted_object.h`](restricted_object.h.md) · [`GameObject.h`](GameObject.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`script_entity_space.h`](script_entity_space.h.md) · [`game_object_space.h`](game_object_space.h.md) · [`space_restriction_manager.h`](space_restriction_manager.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — reached through its declarations in [`patrol_path_manager.h`](patrol_path_manager.h.md); callers name that, not this file.
**Tier floor** — T2: weighted graph traversal with a script callback per waypoint

## Purpose

A patrol route is authored into the level as a graph: named waypoints, each carrying a world
position and a navigation vertex, joined by weighted directed edges. This file is the walker.
It answers one question — *which waypoint next* — and everything else on the page exists to
make that answer respect the creature's restrictors, the route's branch weights, and the
script's right to intervene at every waypoint.

The graph shape matters: a route is not a list. A waypoint may have several outgoing edges,
each with a weight, and the weights are a probability distribution over branches. That is how
one authored route produces varied behaviour across many creatures.

## State

```text
RECORD PatrolPathManager
  route            : optional<PatrolPath>
  route_name       : text
  start_policy     : StartType     # how to join the route
  route_policy     : RouteType     # what to do at a dead end
  actual           : bool
  failed           : bool
  completed        : bool
  random           : bool          # weighted choice, or always the first admissible edge
  current_point    : int           # sentinel when unset
  previous_point   : int           # where we came from; excluded from the next choice
  start_point      : int           # used only by the explicit start policy
  destination      : vector
  extrapolate_hook : optional<script callback>
  restrictors      : the creature's restrictor set
  game_object      : the creature
```

**Invariants** — the previous point is not bookkeeping; it is an *exclusion*. The branch
choice deliberately refuses to go back the way it came, which is what turns a bidirectional
authored graph into forward motion. Without it a creature oscillates between two waypoints.

## `select_point`

**Contract** — the single real operation. Chooses the creature's next waypoint and writes its
navigation vertex to the caller's output. Sets the destination position, marks the manager
fresh and not completed on success, or completed when the route has run out. Fires a script
callback on arrival at each waypoint. Requires a route with at least one waypoint.

The operation has two halves and they run in the same call: *join the route if we are not on
it*, then *step to the next waypoint*.

```text
FUNCTION select_point(position, OUT dest_vertex)
  # ---- half one: join the route, if we are not validly on it ----
  IF NOT actual OR the current point is not a valid waypoint
    chosen := pick_entry_waypoint(position)     # by the start policy; see below
    IF no waypoint was admissible
      dest_vertex := the creature's own navigation vertex     # stand still
      RETURN
    REQUIRE the chosen waypoint's navigation vertex is valid
    IF the previous point is not a valid waypoint
      previous_point := chosen                  # first entry: we came from where we join
    current_point := chosen

    IF the creature is not already standing on the chosen waypoint (within 0.1)
      dest_vertex := chosen waypoint's navigation vertex
      destination := chosen waypoint's position
      actual := true; completed := false
      RETURN                                     # walk to the entry point first

  # ---- half two: step to the next waypoint ----
  fire the script callback "arrived at patrol point" with the current point index

  admissible := outgoing edges of the current waypoint that are
                  not the previous point AND whose target is restrictor-accessible
  IF admissible is empty
    IF route policy is "stop"      THEN completed := true; RETURN
    IF route policy is "continue"
      target := the first restrictor-accessible outgoing edge, ignoring the
                previous-point exclusion
      IF there is none THEN completed := true; RETURN
  ELSE
    target := weighted_choice(admissible)

  previous_point := current_point
  current_point  := target
  dest_vertex    := target waypoint's navigation vertex
  destination    := target waypoint's position
  actual := true; completed := false
```

**Invariants** — the two halves are one call, and the early return between them is what makes
joining a route take a walk rather than a teleport. A creature that is not standing on its
entry waypoint walks to it first, and the branch choice happens on the *next* call, once it
has arrived. The proximity threshold that decides "already standing there" is a tenth of a
metre.

The script callback fires **every time a waypoint is reached**, before the next branch is
chosen, and it is given the creature and the waypoint index. This is the hook the shipped
scripts use to attach behaviour to authored places — say a line, wait, change state. A
rebuild must fire it at exactly this point: after arrival is established and before the next
choice, so that a script may change the route or the policies and have the change take effect
in the same call.

**Notes** — an alternative arrival test is preserved commented-out in the source: it compared
the waypoint's *navigation vertex* to the creature's rather than comparing positions. The
position comparison replaced it because several waypoints can share one navigation vertex —
the mesh is coarser than the authored route — and the vertex test made a creature believe it
had arrived at a waypoint several metres away.

The "no admissible waypoint at all" branch is guarded by an assertion that is deliberately
defeated: the condition is written so that in a checked build a restrictor dump is printed and
the assertion fires, while in a shipping build the creature simply stands where it is. The
source comments it as a concession to a level designer. The shipping behaviour — stand still
rather than fail — is the right one for a rebuild; the diagnostic dump is worth keeping too,
because an inaccessible patrol route is an authoring error and the dump names the restrictors
responsible.

## The entry policies

**Contract** — five ways to choose where to join a route, selected by the start policy.

```text
  first    -> waypoint 0
  last     -> the final waypoint
  nearest  -> the waypoint nearest the given position, among those the
              creature's restrictors admit
  point    -> the explicitly set start waypoint
  next     -> the waypoint after the previous one, if there was a previous one and
              it is admissible; otherwise fall back to "nearest"
```

**Invariants** — the *next* policy is the one with real logic: it takes the waypoint at
index-plus-one if that index exists, and otherwise asks the branch chooser for a successor of
the previous point — so a route whose waypoints are not numbered in walking order still
advances correctly. If the result is inadmissible it falls back to nearest, which is the
universal escape.

The nearest search is given the restrictor test as a predicate rather than filtering
afterwards, so the route's own nearest-point search can reject candidates during its
traversal instead of returning one the creature cannot reach.

Every policy asserts admissibility afterwards, with the restrictor dump attached.

## The weighted branch choice

**Contract** — chooses among the admissible outgoing edges of a waypoint, by edge weight when
randomness is enabled and there is more than one, and by first-admissible otherwise.

```text
FUNCTION weighted_choice(edges of current, excluding previous, restrictor-admissible)
  total := sum of their weights
  threshold := 0
  IF random AND more than one candidate
    threshold := uniform random in [0, total)
  running := 0
  FOR EACH candidate IN the original edge order
    running := running + its weight
    IF running >= threshold THEN RETURN it
```

**Invariants** — a threshold of zero selects the first candidate, since any non-negative
running sum is at least zero. So disabling randomness, or having a single candidate, falls
through to "the first admissible edge in authored order" with no special case. That is why the
two paths share one loop.

The candidate set is walked **twice**: once to count and total the weights, once to select. The
second walk must apply exactly the same two exclusions as the first — not the previous point,
and restrictor-admissible — or the running sum will not correspond to the total and the
selection is biased. A rebuild collecting candidates once into a list avoids the hazard.

The selection is inclusive at the lower bound (`>=`), so a zero-weight edge placed first is
selectable. A rebuild using a strict comparison will make zero-weight edges unreachable, which
changes authored routes that use zero weights as "only if nothing else".

**Notes** — the dead-end handling is where the route policy earns its existence. *Stop* means
the creature has finished and the pipeline reports completion. *Continue* re-runs the edge
scan **without** the previous-point exclusion, so a creature at the end of a dead-end spur
turns around and walks back. That is the difference between a patrol that ends and a patrol
that loops on a linear route.

## `get_next_point`

**Contract** — the same weighted choice applied to an arbitrary waypoint, with the
previous-point exclusion omitted. Returns the sentinel when no edge is admissible.

**Notes** — used only by the *next* entry policy, to find a successor of a previous point when
the plus-one index does not exist. It is the branch chooser with one condition dropped; a
rebuild should factor it into one function with the exclusion as a parameter rather than
duplicating the two-pass loop.

## `extrapolate_path`

**Contract** — answers whether the creature may walk *past* the current waypoint rather than
stopping exactly on it. Asks the script callback if one is installed, and answers true
otherwise.

**Invariants** — true by default. Stopping precisely on every waypoint makes a patrolling
creature visibly hitch at each one; extrapolating lets the detail path curve through. The
script hook exists so that a waypoint with authored business — a door, a ladder, a place to
stand and look — can demand a precise stop.

The answer is consumed by the movement pipeline in two places at once: it selects whether the
level path may extrapolate, and whether the detail path is treated as patrol-style. See
[`movement_manager_patrol.cpp`](movement_manager_patrol.cpp.md).

## `set_previous_point` / `set_start_point`

**Contract** — place the creature on the route explicitly. Each validates that a route is
bound and that the index names a real waypoint, and on failure logs a named script error and
returns without changing anything.

**Invariants** — these are the script-facing setters, so a bad argument is a *mod author's*
error and must be reported as a diagnosable message naming the route and the creature, not a
crash. Every other failure path in this file is an assertion; these two are not, and the
difference is who is at fault.

## `reinit` / `reset` / `path_name`

**Contract** — `reinit` unbinds the route, marks the manager actual and completed, clears the
failure flag and the script callback, and resets. `reset` returns the three point indices and
both policies to their sentinels. `path_name` returns the route's name, or logs a script error
and returns empty text when no route is bound.

**Invariants** — reinitialization leaves the manager *completed*, matching construction: an
unbound manager must not be asked for a next point.
