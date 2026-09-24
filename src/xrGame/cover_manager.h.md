# src/xrGame/cover_manager.h

> Declares the level's cover index, built in [`cover_manager.cpp`](cover_manager.cpp.md) and queried by the template search in [`cover_manager_inline.h`](cover_manager_inline.h.md).

**Needs** — [`quadtree.h`](quadtree.h.md) · [`cover_manager_inline.h`](cover_manager_inline.h.md) · [`cover_point.h`](cover_point.h.md) · [`smart_cover.h`](smart_cover.h.md) · [`xrEngine/profiler.h`](../xrEngine/profiler.h.md)
**Used by** — [`LevelGraphDebugRender.cpp`](LevelGraphDebugRender.cpp.md) · [`base_monster_path.cpp`](ai/monsters/basemonster/base_monster_path.cpp.md) · [`control_path_builder_base_path.cpp`](ai/monsters/control_path_builder_base_path.cpp.md) · [`monster_cover_manager.cpp`](ai/monsters/monster_cover_manager.cpp.md) · [`ai_stalker_cover.cpp`](ai/stalker/ai_stalker_cover.cpp.md) · [`ai_space.cpp`](ai_space.cpp.md) · [`cover_manager.cpp`](cover_manager.cpp.md) · [`cover_manager_inline.h`](cover_manager_inline.h.md) · [`smart_cover_object.cpp`](smart_cover_object.cpp.md) · [`stalker_combat_actions.cpp`](stalker_combat_actions.cpp.md) · [`stalker_combat_planner.cpp`](stalker_combat_planner.cpp.md) · [`stalker_danger_in_direction_actions.cpp`](stalker_danger_in_direction_actions.cpp.md) · [`stalker_danger_property_evaluators.cpp`](stalker_danger_property_evaluators.cpp.md) · [`stalker_get_distance_actions.cpp`](stalker_get_distance_actions.cpp.md) · _and 4 more_
**Tier floor** — T3: a declaration

## Purpose

Declares the cover manager. The build is in [`cover_manager.cpp`](cover_manager.cpp.md); the
search is in [`cover_manager_inline.h`](cover_manager_inline.h.md), which this header pulls
in at the end because the search is parameterised over the caller's evaluator.

Its own load-bearing content is the **shape of the search interface**: the manager does not
know what "good cover" means. It owns the index and the iteration; the caller supplies an
*evaluator* (which scores a point against its own situation) and a *restrictor* (which
vetoes points the caller may not use and weights the rest). The manager supplies a default
restrictor — itself, accepting everything at unit weight — so the two-argument and
three-argument searches are the same search.

Exported units:

- `CCoverManager` — the index plus the search.
- `compute_static_cover` — build the computed cover set from the level's navigation mesh.
- `covers` / `get_covers` — the index, asserting and non-asserting.
- `best_cover` — the search, in two spellings: with a caller restrictor, or with the
  manager's permissive default.
- `add_smart_cover` / `smart_cover` / `smart_covers_storage` — the authored cover layer:
  instantiate one, find one by identifier, reach the description store.
- `clear` — release every cover point; part of level teardown.
- `operator()` / `weight` / `finalize` — the manager's own null restrictor: admit
  everything, weight everything at one, do nothing on selection. These exist so the default
  search has something to pass.
- `edge_vertex` / `cover` / `critical_point` / `critical_cover` — the build's classification
  predicates, protected so a derived manager can change what counts as a cover point.
- `inertia` — private: the rule deciding whether to keep last cycle's answer.
