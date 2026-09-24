# src/xrAICore/Navigation/PatrolPath/patrol_path.h

> Declares a patrol path — a small named graph of authored waypoints with probability-weighted links — and the two enumerations that say how a creature enters and leaves one.

**Needs** — [`../graph_abstract.h`](../graph_abstract.h.md) · [`patrol_point.h`](patrol_point.h.md) · [`patrol_path.cpp`](patrol_path.cpp.md) · [`patrol_path_inline.h`](patrol_path_inline.h.md)
**Used by** — [`patrol_path.cpp`](patrol_path.cpp.md) · [`patrol_path_inline.h`](patrol_path_inline.h.md) · [`patrol_path_params.cpp`](patrol_path_params.cpp.md) · [`patrol_path_params.h`](patrol_path_params.h.md) · [`patrol_path_params_script.cpp`](patrol_path_params_script.cpp.md) · [`patrol_path_storage.cpp`](patrol_path_storage.cpp.md) · [`patrol_path_storage.h`](patrol_path_storage.h.md) · [`patrol_point.cpp`](patrol_point.cpp.md) · [`Artefact.cpp`](../../../xrGame/Artefact.cpp.md) · [`Artefact.h`](../../../xrGame/Artefact.h.md) · [`HelicopterMovementManager.cpp`](../../../xrGame/HelicopterMovementManager.cpp.md) · [`monster_home.cpp`](../../../xrGame/ai/monsters/monster_home.cpp.md) · [`ai_rat.cpp`](../../../xrGame/ai/monsters/rats/ai_rat.cpp.md) · [`ai_rat_templates.cpp`](../../../xrGame/ai/monsters/rats/ai_rat_templates.cpp.md) · _and 7 more_
**Tier floor** — T2: a declaration plus two enumerations whose numeric values are frozen by the script surface.

## Purpose

Declares the type implemented in [`patrol_path.cpp`](patrol_path.cpp.md) and
[`patrol_path_inline.h`](patrol_path_inline.h.md), and owns the two enumerations that are the
vocabulary of patrolling. Those enumerations are substance and live only here: their numeric
values are exported to scripts and appear in shipped game data, so they are frozen.

## State

```text
ENUM PatrolStartType        # where a creature joins a path
  First    = 0              # at the path's first point
  Last     = 1              # at its last
  Nearest  = 2              # at whichever point is closest to the creature
  Point    = 3              # at a point the caller names
  Next     = 4              # at the point after the one the creature last used
  Dummy    = <all bits set> # "not specified"

ENUM PatrolRouteType        # what happens when the path runs out
  Stop     = 0              # remain at the terminal point
  Continue = 1              # keep going, by whatever rule the creature's behaviour says
  Dummy    = <all bits set> # "not specified"
```

**Invariants** — the values are positional and frozen; the script layer exports them by name
(`start`, `stop`, `nearest`, `custom`, `next`, `dummy` for the first; `stop`, `continue`,
`dummy` for the second) and shipped scripts use those names. Note that the script name for
"first" is `start` and for "last" is `stop`, and that `stop` names a *different* value in each
of the two enumerations — the mapping is in
[`patrol_path_params_script.cpp`](patrol_path_params_script.cpp.md) and is not guessable.

The "not specified" member is the saturated value of the underlying integer, the same idiom
this chapter uses for an absent limit. A rebuild should use a genuine absent value.

## Exported units

- construction from a name, and loading from the level editor's raw layout — see
  [`patrol_path.cpp`](patrol_path.cpp.md)
- `point(name)` — the waypoint with that name
- `point(position)` — the waypoint nearest a position
- `point(position, filter)` — the nearest waypoint a caller-supplied predicate accepts
- the inherited small-graph surface: vertices, edges, add and remove, serialize

**Notes** — the path is a graph rather than a list, and its edges carry a **probability**, not a
distance. Nothing in this chapter searches a patrol path; the creature at a waypoint picks an
outgoing edge at random, weighted by those probabilities. That reuse of a graph's weight field
for a wholly different quantity is the one thing about this type a reader must know before the
rest makes sense.
