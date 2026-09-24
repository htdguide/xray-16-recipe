# src/xrAICore/Navigation/PathManagers/path_manager_params_nearest_vertex.h

> The request for "the reachable vertex closest to this point" — a radius plus the point to measure against.

**Needs** — [`path_manager_params.h`](path_manager_params.h.md)
**Used by** — [`path_manager.h`](path_manager.h.md) · [`path_manager_level_nearest_vertex.h`](path_manager_level_nearest_vertex.h.md) · [`path_manager_level_nearest_vertex_inline.h`](path_manager_level_nearest_vertex_inline.h.md)
**Tier floor** — T2: a request record.

## Purpose

A creature frequently wants to go *towards* a position that is not itself on the navigation
mesh — a corpse under an overhang, a point picked by script, a position on the far side of a
gap. This request asks the search to flood outward within a radius and report the single
reachable vertex nearest the target.

## State

```text
RECORD NearestVertexRequest EXTENDS SearchLimits
  target_position        : (real, real, real)  # what "nearest" is measured to; only the
                                               #   horizontal components are compared
  max_range              : real  # RADIUS from the start vertex. Default 6000 = unbounded
  max_iteration_count    : int   # default: no limit
  max_visited_node_count : int   # default: no limit
```

**Invariants** — the target position need not be reachable, on the mesh, or even inside the
level; it is only ever used as the point distances are measured to.

**Notes** — the visited budget defaults to unlimited here while the base record defaults to just
under the vertex pool. That is a real inconsistency: a nearest-vertex search issued with the
default radius and the default budget can exhaust the pool. Every shipped caller passes a real
radius, which bounds it in practice. A rebuild should default this budget the way the base
record does.
