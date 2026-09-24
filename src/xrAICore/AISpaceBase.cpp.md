# src/xrAICore/AISpaceBase.cpp

> Owns the three navigation structures of the running world — the cross-level game graph, the current level's navigation mesh and the patrol paths — together with the one search engine they are all searched through, and refuses to run unless the three were built from the same level.

**Needs** — [`AISpaceBase.hpp`](AISpaceBase.hpp.md) · [`Navigation/game_graph.h`](Navigation/game_graph.h.md) · [`Navigation/level_graph.h`](Navigation/level_graph.h.md) · [`Navigation/game_level_cross_table.h`](Navigation/game_level_cross_table.h.md) · [`Navigation/graph_engine.h`](Navigation/graph_engine.h.md) · [`Navigation/PatrolPath/patrol_path_storage.h`](Navigation/PatrolPath/patrol_path_storage.h.md) · [`Include/xrAPI/xrAPI.h`](../Include/xrAPI/xrAPI.h.md)
**Used by** — [`AISpaceBase.hpp`](AISpaceBase.hpp.md)
**Tier floor** — T2: lifetime and identity checking over data other files own; nothing here touches a byte layout or a device.

## Purpose

Everything else in this chapter is a template or an algorithm with no idea when it should
exist. This file is the lifetime: it decides that a game graph outlives a level, that a
level graph and a search engine do not, and that patrol paths are rebuilt whenever a new
spawn file is read. It is also the place the rest of the engine reaches navigation through
— the object it constructs registers itself in the global environment struct, and every
`GEnv.AISpace` in the codebase is a reach for this instance.

It is a *base*: the game layer derives from it and adds the things that know about
creatures. The split is real — this half has no knowledge of any entity.

## State

```text
RECORD AISpace
  game_graph     : optional<GameGraph>       # the coarse cross-level graph; NOT owned here
  level_graph    : optional<LevelGraph>      # the fine navigation mesh of the loaded level
  graph_engine   : optional<GraphEngine>     # the reusable search workspace
  patrol_paths   : optional<PatrolPathStorage>
```

**Invariants**

- The game graph is *borrowed*. Whoever loads it (the alife simulation) also releases it;
  this record only holds the reference and must be observed to have given it up before it
  is itself destroyed.
- The level graph, the search engine and the patrol storage are owned here and are the
  only things released here.
- At most one search engine exists at a time, and its working capacity is always at least
  the vertex count of the largest graph it will be asked to search. Rebuilding it is how
  capacity changes; there is no growing in place.
- The three structures must agree. A level's navigation mesh, the cross table that maps
  each of its vertices to a game-graph vertex, and the game graph itself each carry an
  identity stamp written when the level was compiled; all three must match or the world is
  refused. A mismatch here is not a graceful degradation — the wrong mesh silently paths
  creatures through walls.

## `Initialize`

**Contract** — brings up the empty, pre-level state: a search engine sized for a token
1024 vertices and an empty patrol registry. Called once, before any level exists. On a
dedicated server it does nothing at all, because a server with no rendering also runs no
navigation. Must not be called when a search engine already exists.

**Notes** — 1024 is a placeholder, not a budget: the engine is destroyed and rebuilt at the
correct size the moment a real graph arrives. Any small number works; what matters is that
a search engine exists so that early callers do not have to test for its absence.

## `SetGameGraph`

**Contract** — attaches or detaches the borrowed cross-level graph. Attaching requires that
none is attached, and rebuilds the search engine sized to the game graph's vertex count.
Detaching requires that one is attached, and destroys the search engine rather than
resizing it — with no graph there is nothing to search.

**Invariants** — attach/detach strictly alternate; the search engine's capacity always
tracks the currently attached graph.

## `Load`

**Contract** — brings the named level's navigation into existence: looks the level up in the
game graph's header by name, creates the level graph for it, tells the game graph which
level is now current, verifies the three identity stamps, and records the level's identifier
in the level graph. Fails loudly — the world does not load — when any stamp disagrees. Blocks
for as long as reading the navigation mesh takes; this is level-load work, not frame work.

```text
FUNCTION load_level(level_name)
  level    <- game_graph.header.level_by_name(level_name)
  level_graph <- LevelGraph.open_current_level()
  game_graph.set_current_level(level.id)

  # the three-way identity check, in the order a mismatch is most likely
  REQUIRE cross_table.header.level_guid == level_graph.header.guid
      # "cross_table doesn't correspond to the AI-map"
  REQUIRE cross_table.header.game_guid  == game_graph.header.guid
      # "graph doesn't correspond to the cross table"

  # the engine must serve whichever of the two graphs is larger, because the
  # same workspace is reused for level searches and game-graph searches
  graph_engine <- GraphEngine(max(game_graph.header.vertex_count,
                                  level_graph.header.vertex_count))

  REQUIRE level.guid == level_graph.header.guid
      # "graph doesn't correspond to the AI-map"

  IF level.name == level_name
    validate(level.id)          # debug-only deep cross-check, see below
  level_graph.level_id <- level.id
```

**Notes** — the search engine is built *between* two of the identity checks purely because
the vertex counts it needs are read from headers already fetched at that point; the order of
the checks themselves carries no meaning beyond producing a useful message first.

The name comparison before the deep validation looks redundant — the level was found by that
name. It guards the case where the header's lookup falls back to a nearby level rather than
failing; the deep check is only meaningful when the level really is the one named.

## `Unload`

**Contract** — releases the current level's navigation. The search engine and level graph go
away. When this is a full unload (not a level-to-level reload) and a game graph is still
attached, a fresh search engine sized to the game graph is left behind, so that off-screen
simulation can keep pathing between levels with no level loaded. Does nothing on a dedicated
server.

**Invariants** — after a reload-unload there is no search engine, because `Load` will build
one; after a plain unload there is one iff a game graph is attached.

## `patrol_path_storage_raw` / `patrol_path_storage`

**Contract** — replaces the patrol registry wholesale from a stream. The *raw* form reads the
level-editor's own layout and snaps every point onto the navigation mesh as it goes, so it
needs the level graph, the cross table and the game graph to do that work; the plain form
reads the already-snapped form written by the spawn compiler and needs nothing else. Both
discard the previous registry first, so patrol paths never accumulate across levels. Both do
nothing on a dedicated server.

**Notes** — two entry points rather than one flag because the two formats have nothing in
common: one is a chunk tree keyed by path name, the other a flat indexed list. See
[`patrol_path_storage.cpp`](Navigation/PatrolPath/patrol_path_storage.cpp.md).

## `cross_table`

**Contract** — the cross table is not a member; it is reached through the game graph, which
owns the table for whichever level is current. Exposed here because callers think of it as a
peer of the two graphs.

## `Validate`

**Contract** — a debug-only deep consistency check between the game graph, the level graph and
the cross table for one level. It is a *diagnostic*, not part of the contract: a shipping
build skips it and the world behaves identically, which is the honest statement that these
invariants are believed rather than enforced at runtime.

```text
FUNCTION validate(level_id)
  REQUIRE level_graph.header.vertex_count == cross_table.header.level_vertex_count

  # every game-graph vertex on this level must sit on a real mesh vertex,
  # that mesh vertex must map back to it, and the vertex's recorded point
  # must lie inside that mesh cell
  FOR EACH gv IN game_graph.vertices WHERE gv.level_id == level_id
    lv <- gv.level_vertex_id
    IF NOT level_graph.valid_vertex_id(lv)
       OR cross_table.vertex(lv).game_vertex_id != index_of(gv)
       OR NOT level_graph.inside(lv, gv.level_point)
      FAIL WITH "Graph doesn't correspond to the cross table"

  # and every spawn point hung off a game vertex must map back to that same vertex
  FOR EACH gv IN game_graph.vertices WHERE gv.level_id == level_id
    FOR EACH sp IN game_graph.spawn_points_of(gv)
      REQUIRE cross_table.vertex(sp.level_vertex_id).game_vertex_id == index_of(gv)
```

**Notes** — the two loops are separable and the second could be folded into the first; they
are kept apart because the first is the one whose failure is reported with a message, while
the second is a bare assertion. The round-trip property being checked in both — mesh vertex
to game vertex and back — is the single thing that makes a cross table trustworthy.
