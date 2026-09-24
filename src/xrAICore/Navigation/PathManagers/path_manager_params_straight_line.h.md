# src/xrAICore/Navigation/PathManagers/path_manager_params_straight_line.h

> The request for a route *and* the length it would have once shortened by line-of-sight — carrying the two endpoints and the answer.

**Needs** — [`path_manager_params.h`](path_manager_params.h.md)
**Used by** — [`path_manager.h`](path_manager.h.md) · [`path_manager_level_straight_line.h`](path_manager_level_straight_line.h.md) · [`path_manager_level_straight_line_inline.h`](path_manager_level_straight_line_inline.h.md)
**Tier floor** — T2: a request record that is also the result slot.

## Purpose

Routes on a navigation mesh follow cell centres and are longer than the line a creature would
actually walk. This request asks for the route's *shortened* length: the length of the polyline
you get by skipping every intermediate cell you can see past. It is used to compare candidate
destinations by real travel distance rather than by mesh hop count.

## State

```text
RECORD StraightLineRequest EXTENDS SearchLimits
  start_point            : (real, real, real)  # the true start, which need not be the
                                               #   centre of the start vertex
  dest_point             : (real, real, real)  # likewise the true destination
  distance               : real                # OUT: the shortened length, written by the
                                               #   policy after the route is built
  max_range              : real  # doubles as the "unreachable" sentinel, see below
  max_iteration_count    : int   # default: no limit
  max_visited_node_count : int   # default: no limit
```

**Invariants** — the request is both input and output. The policy writes the answer back into
`distance`, so the record must outlive the search and must not be shared between concurrent
searches.

**Notes** — `max_range` carries two meanings at once: it is the search's cost bound *and* the
value written into `distance` to mean "the route is longer than you care about". A caller reads
back exactly `max_range` to learn that the answer is "too far", which works only because no
real shortened length ever lands on that value by accident. A rebuild should return an explicit
"beyond the limit" answer instead of overloading the bound.

The two endpoints being separate from the two vertices is the point of the request: the
shortening walk starts from the caller's real position and ends at the caller's real target,
not at the cell centres the search used.
