# src/xrAICore/Navigation/PathManagers/path_manager_level_straight_line.h

> Declares the policy that routes on the mesh and then reports how long that route would be once shortened by line of sight, implemented in [`path_manager_level_straight_line_inline.h`](path_manager_level_straight_line_inline.h.md).

**Needs** — [`path_manager_level.h`](path_manager_level.h.md) · [`path_manager_params_straight_line.h`](path_manager_params_straight_line.h.md) · [`path_manager_level_straight_line_inline.h`](path_manager_level_straight_line_inline.h.md)
**Used by** — [`path_manager.h`](path_manager.h.md) · [`path_manager_level_straight_line_inline.h`](path_manager_level_straight_line_inline.h.md)
**Tier floor** — T2: geometry over the mesh's own visibility query.

## Purpose

Declares the surface implemented in
[`path_manager_level_straight_line_inline.h`](path_manager_level_straight_line_inline.h.md).
The search itself is the ordinary mesh routing policy; everything this file adds happens after
the route exists.

It is built only into the offline tool that compiles navigation data, where route *lengths*
between candidate places are needed in bulk.

## Exported units

- `setup(...)` — as the mesh policy, plus binding the request and seeding its answer with the
  "too far" sentinel
- `create_path(vertex)` — build the route, then measure its line-of-sight-shortened length into
  the request
