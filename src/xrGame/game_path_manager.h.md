# src/xrGame/game_path_manager.h

> Declares the cross-level path manager: a path over the game graph, walked one vertex at a time, with the behaviour in [`game_path_manager_inline.h`](game_path_manager_inline.h.md).

**Needs** — [`abstract_path_manager.h`](abstract_path_manager.h.md) · [`game_path_manager_inline.h`](game_path_manager_inline.h.md) · [`xrAICore/Navigation/game_graph.h`](../xrAICore/Navigation/game_graph.h.md)
**Used by** — [`base_monster_script.cpp`](ai/monsters/basemonster/base_monster_script.cpp.md) · [`game_path_manager_inline.h`](game_path_manager_inline.h.md) · [`movement_manager.cpp`](movement_manager.cpp.md) · [`movement_manager_game.cpp`](movement_manager_game.cpp.md) · [`script_game_object3.cpp`](script_game_object3.cpp.md)
**Tier floor** — T3: a declaration over the generic path manager

## Purpose

The game-graph specialization of the generic path manager. A creature's movement is planned
at two scales — across levels on the game graph, and within a level on the level graph — and
this is the coarse half. It holds a path of graph vertices and an index into it, and answers
"where next" and "are we there".

Substance is in [`game_path_manager_inline.h`](game_path_manager_inline.h.md); the split is
an artifact of the original being a template.

Exported units:

- `CBaseGamePathManager` — the manager. Final: nothing specializes it further.
- `reinit` — plain delegation to the generic manager, bound to a game graph.
- `actual` — whether the held path still starts where the creature actually is.
- `select_intermediate_vertex` — advance to the next vertex of the path.
- `completed` — whether the path has been walked.
- `before_search` / `after_search` — the generic manager's two hooks, both deliberately
  empty here. The game graph needs no per-search setup or teardown; the level-graph
  specialization does, which is why the hooks exist at all.
