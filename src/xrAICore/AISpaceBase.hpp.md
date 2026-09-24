# src/xrAICore/AISpaceBase.hpp

> Declares the navigation-ownership surface implemented in [`AISpaceBase.cpp`](AISpaceBase.cpp.md), and the split between what the game layer may call and what only the level lifecycle may.

**Needs** — [`AISpaceBase.cpp`](AISpaceBase.cpp.md)
**Used by** — [`AISpaceBase.cpp`](AISpaceBase.cpp.md) · [`patrol_path_params.cpp`](Navigation/PatrolPath/patrol_path_params.cpp.md) · [`patrol_point.cpp`](Navigation/PatrolPath/patrol_point.cpp.md) · [`game_graph_script.cpp`](Navigation/game_graph_script.cpp.md) · [`graph_engine_inline.h`](Navigation/graph_engine_inline.h.md) · [`pch.hpp`](pch.hpp.md) · [`chimera_attack_state_inline.h`](../xrGame/ai/monsters/chimera/chimera_attack_state_inline.h.md) · [`ai_space.cpp`](../xrGame/ai_space.cpp.md) · [`ai_space.h`](../xrGame/ai_space.h.md)
**Tier floor** — T2: a declaration of ownership and visibility; nothing here computes.

## Purpose

Declares the type whose substance lives in [`AISpaceBase.cpp`](AISpaceBase.cpp.md). Its one
load-bearing decision is the *visibility split*: the lifecycle operations are reachable only
by the derived game-layer type, while the accessors are public. Anyone may ask where the
level graph is; only the thing that owns level loading may unload it.

The type is designed to be inherited, not instantiated: the derived type in the game layer
adds the creature-aware parts. A rebuild may equally make this a component the game layer
holds rather than a base it extends — nothing here depends on the inheritance.

## Exported units

Lifecycle, visible only to the derived type:

- `Load(level_name)` — bring up one level's navigation
- `Unload(reload)` — tear it down, keeping a cross-level search engine if this is not a reload
- `Initialize()` — the pre-level state
- `SetGameGraph(graph)` — attach or detach the borrowed cross-level graph
- `Validate(level_id)` — debug-only deep consistency check
- `patrol_path_storage(stream)` / `patrol_path_storage_raw(stream)` — replace the patrol registry

Accessors, public:

- `game_graph()` / `get_game_graph()` — the coarse cross-level graph; the first form asserts
  it exists, the second admits it may not
- `level_graph()` / `get_level_graph()` — the current level's navigation mesh, same pairing
- `cross_table()` / `get_cross_table()` — the mesh-vertex-to-game-vertex map for the current level
- `patrol_paths()` — the patrol registry
- `graph_engine()` — the shared search workspace

**Notes** — the doubled accessors are the whole of the null-handling policy in this module:
callers that cannot proceed without a structure use the asserting form and get a crash at the
point of the mistake; callers that legitimately run with no level loaded use the optional
form. A rebuild expresses this as one accessor returning an optional plus one that unwraps it.
