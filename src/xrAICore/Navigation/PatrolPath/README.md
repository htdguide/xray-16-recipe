# src/xrAICore/Navigation/PatrolPath — authored waypoint graphs

> The one part of this chapter that a level designer wrote by hand: named routes of
> waypoints with probability-weighted links, registered per level and handed to creatures by
> name.

Part of [chapter 14](../../README.md).

## What this directory is responsible for

A **patrol path** is a small named graph. Its vertices are authored waypoints — a position,
a name, a flag word the game layer interprets, and the navigation identities the position
snaps to. Its edges carry a probability weight, so a route can fork and a creature picks a
branch by weighted draw rather than always walking the same circuit.

The directory owns four things: the path graph, the waypoint, the per-level registry that
loads and owns every path on a level, and the small handle a creature or a script holds onto
one. The registry is built at level load; the handle is what everything above the chapter
actually touches.

Nothing here decides *behaviour*. What a creature does when it reaches a waypoint — wait,
look, play an animation — is the flag word plus the game layer's interpretation of it, in
chapter 24 and in the shipped scripts.

## Where it sits

It rests on the core layer's chunked container format and interned strings, and on
[`Navigation/`](../README.md) for the two graphs a waypoint snaps onto. Its consumers are the
smart-terrain job system and the scripted movement layer, both in chapter 23/24, and the
script surface directly — `patrol` is one of the handful of engine types mods build their own
routes with.

## The load-bearing ideas

**A waypoint is stored as a position and used as a graph vertex.** The authored position is
a world coordinate; what a creature can actually walk to is a mesh cell. The conversion
happens once, at load, and the waypoint keeps both. A position that resolves to no cell is a
level-authoring error and must be reported as one, not silently accepted — a creature sent to
an unreachable waypoint simply stops.

**The path graph is read field by field, not aliased.** Unlike the navigation graphs, a patrol
path is small and is parsed out of a chunk stream rather than pointed at. Its serialized form
is the engine's own, versioned only against itself, so a rebuild is free to change it as long
as it still reads what the shipped levels contain.

**Two shipped forms of the same data.** A level's paths arrive either as the level editor's
authored chunk tree or as the pre-converted form the spawn compiler writes. The registry
accepts both; the difference is load-time cost, not content.

**Lookup by name is the whole interface.** The registry is ordered by interned name and
searched binary; a creature holds a resolved reference, not a name, once it has one. The
caller chooses whether a missing path is a failure or an empty answer, and the choice differs
by call site — a script asking for a path that may not exist is not the same event as a smart
terrain's job referring to one that must.

**Entry and exit are enumerations with frozen values.** How a creature joins a path and how
it leaves one are named constants exposed to scripts, so their numeric values are part of the
frozen script surface (conformance criterion 10), not an internal detail.

## The twins

| Twin | Role |
|---|---|
| [`patrol_path.h`](patrol_path.h.md) | A patrol path: a small named graph of authored waypoints with probability-weighted links, plus the entry and exit enumerations |
| [`patrol_path.cpp`](patrol_path.cpp.md) | Reading one authored path out of the level editor's layout into that graph |
| [`patrol_path_inline.h`](patrol_path_inline.h.md) | The three ways to find a waypoint: by name, by proximity, and by proximity among those a caller will accept |
| [`patrol_point.h`](patrol_point.h.md) | One waypoint: a named position snapped onto the navigation graphs, with a flag word the game interprets |
| [`patrol_point.cpp`](patrol_point.cpp.md) | Loading a waypoint, and the conversion that turns an editor position into a cell a creature can stand on |
| [`patrol_point_inline.h`](patrol_point_inline.h.md) | Field reads, each guarded by the requirement that the point has actually been loaded |
| [`patrol_path_storage.h`](patrol_path_storage.h.md) | The per-level registry of every patrol path, keyed by name |
| [`patrol_path_storage.cpp`](patrol_path_storage.cpp.md) | Building the registry from either shipped form, and owning every path in it |
| [`patrol_path_storage_inline.h`](patrol_path_storage_inline.h.md) | Name lookup, with the caller choosing whether absence is a failure or an answer |
| [`patrol_path_params.h`](patrol_path_params.h.md) | The handle a creature or a script holds onto a named path |
| [`patrol_path_params.cpp`](patrol_path_params.cpp.md) | Resolving the path once, then answering every waypoint question by index |
| [`patrol_path_params_inline.h`](patrol_path_params_inline.h.md) | Empty — the handle has no inline surface left |
| [`patrol_path_params_script.cpp`](patrol_path_params_script.cpp.md) | Publishing the handle to scripts as `patrol`, with the two enumerations |

## What could not be recovered

- The **fifteen-centimetre lift** applied to a waypoint's position before it is matched to a
  mesh cell. Its effect is clear — a point authored flush with the floor must not resolve to
  whatever lies beneath it — but nothing derives the number.
- The two enumerations **collide in the script namespace**: `stop` and `dummy` are each
  registered twice with different meanings, and which registration survives depends on the
  binding layer's ordering. The shipped scripts depend on whichever wins, so this must be
  checked against a running original rather than reasoned out.
- The registry permits **two keys aliasing one owned path** while teardown frees every entry.
  No shipped level takes that route, so it has never surfaced; whether aliasing was meant to
  coexist with teardown at all is unclear.
- A **named constant holding one particular path's name** sits unused in the source. A
  debugging leftover with no recoverable purpose.
