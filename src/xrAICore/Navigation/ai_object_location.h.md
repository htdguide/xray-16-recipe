# src/xrAICore/Navigation/ai_object_location.h

> An entity's position expressed as navigation, not as coordinates — which fine mesh vertex and which coarse game vertex it occupies.

**Needs** — [`ai_object_location_inline.h`](ai_object_location_inline.h.md) · [`ai_object_location_impl.h`](ai_object_location_impl.h.md) · [`game_graph_space.h`](game_graph_space.h.md) · [`level_graph_space.h`](level_graph_space.h.md)
**Used by** — [`ai_object_location_impl.h`](ai_object_location_impl.h.md) · [`ai_object_location_inline.h`](ai_object_location_inline.h.md) · [`GameObject.cpp`](../../xrGame/GameObject.cpp.md) · [`Missile.cpp`](../../xrGame/Missile.cpp.md) · [`RocketLauncher.cpp`](../../xrGame/RocketLauncher.cpp.md) · [`ai_monster_squad_rest.cpp`](../../xrGame/ai/monsters/ai_monster_squad_rest.cpp.md) · [`control_direction.cpp`](../../xrGame/ai/monsters/control_direction.cpp.md) · [`control_path_builder.cpp`](../../xrGame/ai/monsters/control_path_builder.cpp.md) · [`control_path_builder_base.cpp`](../../xrGame/ai/monsters/control_path_builder_base.cpp.md) · [`control_path_builder_base_path.cpp`](../../xrGame/ai/monsters/control_path_builder_base_path.cpp.md) · [`controller_state_attack_camp_inline.h`](../../xrGame/ai/monsters/controller/controller_state_attack_camp_inline.h.md) · [`state_look_unprotected_area.h`](../../xrGame/ai/monsters/states/state_look_unprotected_area.h.md) · [`artefact_activation.cpp`](../../xrGame/artefact_activation.cpp.md) · [`detail_path_manager.cpp`](../../xrGame/detail_path_manager.cpp.md) · _and 19 more_
**Tier floor** — T2: two identities and their validity rules. It sits inside entities that are themselves T1, but nothing here demands it.

## Purpose

Every entity that the AI reasons about carries one of these. It is the answer to "where is this
thing" in the only terms the AI can use: a mesh vertex, for everything inside the loaded level,
and a game vertex, for everything the off-screen simulation moves around. An entity outside the
loaded level has a meaningful game vertex and no meaningful mesh vertex, which is why the two are
tracked independently and each can be invalid on its own.

This is not a copy of the entity's transform. It is derived from it, refreshed as the entity
moves, and it is what the AI reads — a creature asks "which vertex is my enemy on", never "what
are my enemy's coordinates", because paths are computed in vertices.

## State

```text
RECORD ObjectLocation
  level_vertex_id : int          # the fine mesh vertex; invalid when off the loaded level
  game_vertex_id  : int (16-bit) # the coarse cross-level vertex; invalid before placement
```

**Invariants** — both fields are always either a valid identity of their graph or that graph's
designated invalid value, never uninitialized. The invalid values differ in width — the game
graph's identity is sixteen bits — which is why each graph is asked for its own invalid value
rather than one being hard-coded here. The two are not required to agree: the cross table says
which game vertex covers a given mesh vertex, but nothing here enforces that the two fields
satisfy it, and the game deliberately lets them drift for entities in transit between levels.

## `init` / `reinit`

**Contract** — set both identities to invalid. Asks each graph for its own invalid value when that
graph is loaded, and falls back to an all-ones value of the right width when it is not — because
an entity may be constructed before a level's mesh exists, during save loading or during alife
startup.

```text
FUNCTION init()
  IF the level mesh is loaded  level_vertex_id <- mesh.invalid_vertex()
  ELSE                         level_vertex_id <- all_ones(32)
  IF the game graph is loaded  game_vertex_id  <- graph.invalid_vertex()
  ELSE                         game_vertex_id  <- all_ones(16)
```

**Notes** — asking the graph rather than writing the constant is not ceremony: the graph asserts
that the value it returns does not validate, so a mesh whose vertex count somehow reached the
full identity range is caught at the point where an entity would otherwise be silently placed on
a real vertex. Construction runs `init`, and `reinit` is the same operation under the name the
entity lifecycle uses when an entity is recycled.

## `level_vertex_id` / `game_vertex_id` — reading

**Contract** — return the stored identity. No validation: a caller is entitled to hold an invalid
location and test it. These are the cheapest possible reads and they are on every AI decision's
path.

## `level_vertex` / `game_vertex` — reading as records

**Contract** — return the graph record the identity names. Requires the identity to be valid;
asking for the record of an invalid location is a programming error. Implemented in
[`ai_object_location_impl.h`](ai_object_location_impl.h.md), because it needs the full graph
definitions.

## `level_vertex` / `game_vertex` — writing

**Contract** — set the identity, either directly or by handing over a graph record, from which the
identity is derived. Both forms validate against the graph: writing an identity the graph does not
recognize is a programming error, caught immediately rather than at the next path search.

**Notes** — accepting a record and deriving the identity from it exists because most callers have
just obtained a record from a query and would otherwise convert back and forth. The derivation is
pointer arithmetic against the graph's array base, which means a record from a *different* graph
produces a plausible-looking wrong identity; the validity assertion catches most but not all of
those. A rebuild with typed handles avoids the whole class.
