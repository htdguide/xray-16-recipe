# src/xrAICore/Navigation/PatrolPath/patrol_point.cpp

> One authored waypoint, and the conversion that turns an editor's position into a navigation-mesh vertex a creature can actually stand on.

**Needs** — [`patrol_point.h`](patrol_point.h.md) · [`../level_graph.h`](../level_graph.h.md) · [`../game_level_cross_table.h`](../game_level_cross_table.h.md) · [`../game_graph.h`](../game_graph.h.md) · [`patrol_path.h`](patrol_path.h.md) · [`../../AISpaceBase.hpp`](../../AISpaceBase.hpp.md) · [`../../../Common/object_broker.h`](../../../Common/object_broker.h.md)
**Used by** — [`patrol_point.h`](patrol_point.h.md)
**Tier floor** — T2: field-by-field stream reads and a mesh lookup; no memory-image aliasing.

## Purpose

A level designer places a waypoint by dragging it in a viewport. What comes out is a position in
world space, which is not something a creature can be sent to — creatures move between navigation
mesh vertices. This file is that conversion, and the two decisions in it are how a position
becomes a vertex and what to do when the two disagree.

The conversion happens **once, at load**, not per query. Everything after load is a stored
identifier.

## State

```text
RECORD PatrolPoint
  name            : text     # authored; unique within a path by convention only
  position        : (real, real, real)   # authored, then possibly snapped — see below
  flags           : int (32-bit)         # authored bits; meanings belong to the game layer
  level_vertex_id : int      # navigation-mesh vertex; the invalid marker when off-mesh
  game_vertex_id  : int      # cross-level graph vertex; derived from the mesh vertex
  path            : ref to PatrolPath    # debug builds only, for diagnostics
```

**Invariants** — after loading, either the mesh vertex is valid and the position lies inside that
vertex's cell, or the mesh vertex is the invalid marker. There is no third state. The cross-level
vertex is always derived from the mesh vertex through the cross table, never authored.

## `load_raw`

**Contract** — reads one waypoint from the level editor's authored form: position, flag word,
name — in that order, all fixed-width except the name, which is zero-terminated. Then resolves
the mesh vertex and snaps the position. Tolerates the absence of a level graph entirely, which is
how the same reader serves a tool that has no mesh loaded.

```text
FUNCTION load_raw(level_graph, cross_table, game_graph, stream)
  position <- read three 32-bit reals
  flags    <- read 32-bit integer
  name     <- read zero-terminated text
  IF level_graph exists AND position is within the mesh's bounds
    level_vertex_id <- level_graph.vertex_containing(position raised by 0.15)
  ELSE
    level_vertex_id <- invalid
  correct_position(level_graph, cross_table, game_graph)
```

**Notes** — **the 0.15 lift is load-bearing and undocumented.** The lookup point is raised by
fifteen centimetres before the mesh is asked which cell contains it. Waypoints are authored flush
with the floor, and the mesh's own cells are surfaces; without the lift a point exactly on a
surface resolves ambiguously, and on a floor with anything beneath it — a walkway, a stair, a
second storey — it can resolve to the cell below. Fifteen centimetres is small enough to stay
under any ceiling and large enough to clear authoring imprecision. The number itself has no
derivation in the source; a rebuild should keep it and expect to retune it only if the mesh's
vertical tolerance changes.

Reading three reals then an integer then a string is the frozen order; the format is little-endian
throughout and a big-endian rebuild must swap at each field.

## `correct_position`

**Contract** — reconciles the authored position with the vertex it resolved to. Does nothing when
there is no level graph, when the position is outside the mesh's bounds, or when the vertex is
invalid. Otherwise: if the position does not actually lie inside the resolved vertex's cell, the
position is *replaced* by that cell's own position. Then the cross-level vertex is read out of the
cross table.

**Invariants** — this is what establishes the invariant above. A waypoint's position always lies
in its own cell afterwards, so a creature sent to the waypoint's position and a creature sent to
its mesh vertex arrive at the same place.

**Notes** — the authored position is *overwritten*, not adjusted. A waypoint a designer placed
slightly off the walkable surface is moved to the cell centre, which can be up to half a cell
away. That is the right trade — a waypoint a creature cannot reach is worse than one moved a
little — but it means the shipped position is not always the authored one, and a rebuild that
preserves the authored position instead will place creatures where they cannot stand.

## `load` / `save`

**Contract** — the pre-converted runtime form: name, position, flags, mesh vertex, cross-level
vertex, in that order, symmetrically. No snapping happens on this path: the values were resolved
when the level was compiled and are taken as given.

**Invariants** — the two orders must match, and must match what the spawn compiler wrote. This is
the form the engine actually reads at runtime.

## `level_vertex_id()` / `game_vertex_id()` — the no-argument forms

**Contract** — the same two identifiers, for callers that do not have the navigation structures to
hand. Each checks whether the waypoint's cross-level vertex belongs to the *currently loaded*
level: if it does, the identifier is re-read through the loaded structures, which re-validates it;
if it does not, the stored value is returned unchanged.

```text
FUNCTION level_vertex_id() -> int
  IF game_graph.vertex(game_vertex_id).level_id == loaded_level.id
    RETURN level_vertex_id checked against the loaded mesh
  RETURN level_vertex_id            # another level's mesh is not loaded; nothing to check against
```

**Notes** — this is the whole cross-level story for waypoints. A patrol path on a level that is not
loaded still has meaningful cross-level vertices — the off-screen simulation moves entities along
them — but its mesh vertices cannot be validated, because that mesh is not in memory. The check
exists so that validation happens exactly where it is possible and is skipped where it is not,
rather than being skipped everywhere or crashing on absent data.

These two reach the navigation structures through the global environment struct. Every other
routine here takes them as arguments; these cannot, because their callers are script-facing and
have none. A rebuild should inject them.

## `verify_vertex_id` (debug builds)

**Contract** — asserts that a waypoint's mesh vertex is valid, naming the waypoint and its path in
the message. Called from every accessor in debug builds.

**Notes** — the diagnostic is the reason waypoints keep a back-reference to their path at all. A
waypoint that failed to land on the mesh is a content error, and the message is how a level
designer finds which one; without the path name the message is useless, because waypoint names
repeat across paths.

## Equality

**Contract** — declared and unconditionally fails. The small-graph template this waypoint is stored
in requires its payload to be comparable; nothing ever compares two waypoints. A rebuild whose
container does not demand equality should omit it rather than reproduce a routine that exists to
be never called.
