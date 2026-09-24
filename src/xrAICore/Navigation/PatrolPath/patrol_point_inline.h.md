# src/xrAICore/Navigation/PatrolPath/patrol_point_inline.h

> The patrol point's field reads, each guarded by the requirement that the point has actually been loaded.

**Needs** — [`patrol_point.h`](patrol_point.h.md)
**Used by** — [`patrol_point.h`](patrol_point.h.md)
**Tier floor** — T2: field access.

## Purpose

The accessor half of the patrol point; the substance is in
[`patrol_point.cpp`](patrol_point.cpp.md).

## State

Stateless. The record is in [`patrol_point.h`](patrol_point.h.md).

## Exported units

- `position()` / `flags()` / `name()` — field reads.
- `level_vertex_id(mesh, cross, graph)` / `game_vertex_id(mesh, cross, graph)` — identity reads
  with the graphs passed explicitly. The graph arguments are used only to re-check the point
  against them; the returned value is the stored field either way.
- `path(owning_path)` — record which route this point belongs to, for diagnostics.
- equality — declared and deliberately unimplemented: reaching it is a hard error.

**Invariants** — every accessor requires the point to have been loaded. A patrol point exists in
an unloaded state between construction and its first load, and reading a field then would return
whatever the authored data has not yet supplied. This is checked in debug builds only, which
means the invariant is real but unenforced in a shipping build.

**Notes** — equality is declared so that the containers holding patrol points compile, and made a
hard error so that nothing silently relies on it. A rebuild should simply not declare it.

The identity accessors' validity check asserts that a point on the loaded level resolves to a
valid mesh vertex, and reports the point and route by name when it does not. That diagnostic is
the only thing that makes a misplaced waypoint findable in a level of several hundred of them, so
it is worth keeping even though it is debug-only.
