# src/xrAICore/Navigation/PatrolPath/patrol_path_params.cpp

> Resolves a patrol path by name once and then answers every question a creature or a script asks about its waypoints by index.

**Needs** — [`patrol_path_params.h`](patrol_path_params.h.md) · [`patrol_path_storage.h`](patrol_path_storage.h.md) · [`patrol_path.h`](patrol_path.h.md) · [`../../AISpaceBase.hpp`](../../AISpaceBase.hpp.md) · [Seam: Script virtual machine](../../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`patrol_path_params.h`](patrol_path_params.h.md)
**Tier floor** — T2: indexed lookups into a small graph.

## Purpose

Scripts and creature behaviours talk about patrol paths by name and about waypoints by index.
The path itself is a graph keyed by vertex identifier. This file is the adapter, and its one
structural decision is that the name is resolved **once, at construction**, and hard-fails if
the path does not exist — so every later question is a direct lookup and no caller has to handle
a missing path.

## State

```text
RECORD PatrolPathParams
  path            : ref to PatrolPath   # BORROWED from the registry; resolved at construction
  path_name       : text                # kept for diagnostics
  start_type      : PatrolStartType     # default: nearest
  route_type      : PatrolRouteType     # default: continue
  random          : bool                # default: true — pick links by weight rather than in order
  previous_index  : int                 # the waypoint last used, for the "next" start type;
                                        #   defaults to the invalid marker
```

**Invariants** — the path reference is non-null for the object's whole life: construction with a
name the registry does not hold is a hard failure naming the path. The path is borrowed and must
not outlive the registry, which means a handle must not survive a level change.

**Notes** — the defaults are the common case in shipped data: join at the nearest waypoint, keep
going past the end, choose links at random. A caller that supplies nothing gets the behaviour
most creatures want.

## Construction

**Contract** — looks the name up in the level's patrol registry, tolerating absence at the lookup
so that the failure can be raised here with the name in it. Stores the four settings verbatim.
Allocates nothing.

## `count`

**Contract** — the number of waypoints.

## `point(index)`

**Contract** — a waypoint's position. An index the path does not hold is *recovered from*, not
asserted: the mistake is reported to the script log with the index and the path name, and the
first waypoint is substituted. Asserts that the path is non-empty.

**Notes** — this is the only routine here that recovers. Waypoint indices reach this from script
arithmetic, where an off-by-one is a script bug that should not take the process down mid-game;
every other accessor is reached from engine code where a bad index is a real error. The
substitution is a deliberate "keep playing, tell the modder" choice, and a rebuild should keep
both halves of it — silently substituting without logging turns a script bug into a mystery.

## `level_vertex_id(index)` / `game_vertex_id(index)`

**Contract** — the waypoint's navigation-mesh vertex and its cross-level graph vertex. Both go
through the waypoint's own resolution, which re-validates against the loaded level when the
waypoint is on it. See [`patrol_point.cpp`](patrol_point.cpp.md).

## `point(name)`

**Contract** — the index of the waypoint with that authored name, or the invalid marker when
there is none.

**Notes** — this scans the path twice on a hit: once to find the waypoint and once more to read
its identifier. Harmless at these sizes and worth not reproducing.

## `point(position)`

**Contract** — the index of the waypoint nearest a position.

**Invariants** — the path must be non-empty. Unlike the name lookup, there is no "not found"
answer, and none is checked for: an empty path faults here.

## `flag(index, bit)` / `flags(index)`

**Contract** — one authored flag bit of a waypoint, and the whole flag word. The bits are
authored in the level editor and their meanings belong to the game layer, not here; this file
only carries them through.

## `name(index)`

**Contract** — the waypoint's authored name.

## `terminal(index)`

**Contract** — whether the waypoint has no outgoing links, which is what makes it an end of the
path. This is what the "what happens when the path runs out" setting is tested against.

**Invariants** — terminality is a property of the *graph*, not an authored flag: a waypoint is
terminal exactly when nothing leads onward from it. A rebuild must not cache it, because the
links come from authored data and a path can have several terminals or none at all — a cycle has
none, and a cycle is a normal patrol route.
