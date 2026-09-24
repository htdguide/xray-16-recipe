# src/xrAICore/Navigation/PatrolPath/patrol_path_inline.h

> The three ways to find a waypoint in a patrol path — by name, by proximity, and by proximity among those a caller will accept.

**Needs** — [`patrol_path.h`](patrol_path.h.md) · [`patrol_point.h`](patrol_point.h.md)
**Used by** — [`patrol_path.h`](patrol_path.h.md)
**Tier floor** — T2: linear scans over a handful of waypoints.

## Purpose

A creature joining a patrol path has to be told which waypoint to start at, and the three
answers it might be given — "the one called X", "the nearest one", "the nearest one that is
also Y" — are here. All three are linear scans, and that is a deliberate acceptance: authored
paths hold single-digit numbers of waypoints, and the lookups run when a creature adopts a path
rather than per frame.

## `point(name)`

**Contract** — the first waypoint whose authored name matches, or nothing. Linear scan in vertex
order. Names are not required to be unique within a path and the first match wins.

## `point(position, filter)`

**Contract** — the waypoint nearest the given position among those the filter accepts, or
nothing when the filter accepts none. Distance is the full three-dimensional squared distance
between waypoint position and the given position; no square root is taken, since only the
comparison matters.

```text
FUNCTION nearest(position, filter) -> optional<Waypoint>
  best <- none ; best_d <- infinity
  FOR EACH waypoint IN vertices
    IF NOT filter(waypoint.position) THEN CONTINUE
    d <- squared_distance(waypoint.position, position)
    IF d < best_d THEN best_d <- d ; best <- waypoint
  RETURN best
```

**Notes** — the filter takes only a position, not the waypoint. That is the limit of what callers
have needed — "is this spot somewhere I am allowed to be" — and it keeps the predicate
independent of the waypoint type. A rebuild passing the whole waypoint loses nothing.

Ties go to the earlier waypoint in vertex order, because the comparison is strict. Authored
paths do put two waypoints at the same spot occasionally, so the tie-break is observable.

## `point(position)`

**Contract** — the nearest waypoint with no filtering. Implemented as the filtered form with a
predicate that accepts everything, so the two cannot drift apart.

## `name` (debug builds)

**Contract** — sets the path's name once, on a path that does not yet have one, from a non-empty
string. Both conditions are asserted. Present so that the storage can stamp the registry key
onto the path after deserializing it; the name is otherwise only ever used in diagnostics.
