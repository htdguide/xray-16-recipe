# src/xrGame/alife_smart_terrain_task_inline.h

> Construction of a job destination: five spellings collapsing to two forms.

**Needs** — [`alife_smart_terrain_task.h`](alife_smart_terrain_task.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/game_graph.h`](../xrAICore/Navigation/game_graph.h.md)
**Used by** — [`alife_smart_terrain_task.h`](alife_smart_terrain_task.h.md)
**Tier floor** — T2: field initialization with one validity check

## Purpose

Holds the constructors, which carry the one real invariant of the type. Resolution is in
[`alife_smart_terrain_task.cpp`](alife_smart_terrain_task.cpp.md).

## The authored form

**Contract** — from a patrol path name and a point index (zero when omitted). Fails hard
if the path or point does not exist.

```text
FUNCTION construct(path_name, point_index)
  record path_name and point_index          # diagnostic builds only
  patrol_point    = none
  game_vertex_id  = unset
  level_vertex_id = unset
  setup_patrol_point(path_name, point_index)
```

**Invariants** — **both vertex fields are explicitly left unset before the point is
bound.** That is not defensive initialization: it is how the accessors later know which
form they are in. A rebuild with an explicit two-shape type gets this for free.

## The computed form

**Contract** — from a game-graph vertex and a level-graph vertex. Fails hard if the game
vertex is not a valid vertex of the loaded game graph, with the offending value in the
message.

```text
FUNCTION construct(game_vertex, level_vertex)
  REQUIRE game_graph.is_valid_vertex(game_vertex)
  game_vertex_id  = game_vertex
  level_vertex_id = level_vertex
```

**Invariants** — the coarse vertex is validated at construction; the fine vertex is not,
because it may legitimately refer to a level that is not loaded and cannot be checked
here. That asymmetry is correct and a rebuild should keep it. The patrol-point field is
left holding whatever it held — in practice nothing, since the object is freshly
constructed — which is harmless only because the accessors never consult it in this form.
A rebuild should clear it anyway.

## `patrol_point`

**Contract** — the bound patrol point, required to be present. Only the authored form's
accessors call it.
