# src/xrGame/movement_manager.h

> Declares the movement manager — the three-level path pipeline every walking creature moves through — implemented across [`movement_manager.cpp`](movement_manager.cpp.md) and its four siblings.

**Needs** — [`ai_monster_space.h`](ai_monster_space.h.md) · [`movement_manager_space.h`](movement_manager_space.h.md) · [`movement_manager_inline.h`](movement_manager_inline.h.md) · [`xrAICore/Navigation/graph_engine_space.h`](../xrAICore/Navigation/graph_engine_space.h.md) · [`xrAICore/Navigation/game_graph_space.h`](../xrAICore/Navigation/game_graph_space.h.md)
**Used by** — [`CustomMonster.cpp`](CustomMonster.cpp.md) · [`CustomMonster_VCPU.cpp`](CustomMonster_VCPU.cpp.md) · [`control_path_builder.cpp`](ai/monsters/control_path_builder.cpp.md) · [`control_path_builder.h`](ai/monsters/control_path_builder.h.md) · [`ai_rat_animations.cpp`](ai/monsters/rats/ai_rat_animations.cpp.md) · [`ai_rat_templates.cpp`](ai/monsters/rats/ai_rat_templates.cpp.md) · [`rat_state_initialize.cpp`](ai/monsters/rats/rat_state_initialize.cpp.md) · [`detail_path_builder.h`](detail_path_builder.h.md) · [`enemy_manager.cpp`](enemy_manager.cpp.md) · [`item_manager.cpp`](item_manager.cpp.md) · [`level_path_builder.h`](level_path_builder.h.md) · [`movement_manager.cpp`](movement_manager.cpp.md) · [`movement_manager_game.cpp`](movement_manager_game.cpp.md) · [`movement_manager_inline.h`](movement_manager_inline.h.md) · _and 8 more_
**Tier floor** — T3: a declaration

## Purpose

Declares `CMovementManager`, the component a creature owns that turns "go there" into a
per-frame position. Substance is spread across five implementation twins because the file
was split by *path level*, not by concern:
[`movement_manager.cpp`](movement_manager.cpp.md) (lifecycle, the state driver, prediction),
[`movement_manager_game.cpp`](movement_manager_game.cpp.md) (cross-level paths),
[`movement_manager_level.cpp`](movement_manager_level.cpp.md) (within-level paths),
[`movement_manager_patrol.cpp`](movement_manager_patrol.cpp.md) (authored patrol paths) and
[`movement_manager_physic.cpp`](movement_manager_physic.cpp.md) (the actual per-frame motion).

Exported units:

- `CMovementManager` — owns eleven subordinates: the two search-cost evaluators, the game
  location selector, the game path manager, the level path manager, the detail path
  manager, the patrol path manager, the restrictor set, the location manager, and the two
  background path builders. Each stage feeds the next.
- `set_path_type` / `path_type` — choose which of the four pipelines runs.
- `set_game_dest_vertex` / `set_level_dest_vertex` — name a destination on the coarse or
  the fine graph; either invalidates the path.
- `update_path` — advance the path state machine one step.
- `on_frame` — the per-frame entry: advance the pipeline, then move.
- `move_along_path` — consume the detail path against the physics character.
- `actual` / `actual_all` / `path_completed` — the three freshness questions.
- `enable_movement` / `enabled` — suspend motion without losing the path.
- `set_desirable_speed` / `old_desirable_speed` / `speed` — commanded speed versus achieved speed.
- `set_body_orientation` / `body_orientation` — the head/torso rotation the path-follower
  reads to decide which way the creature is facing.
- `path` — the final detail path as travel points.
- `predict_position` / `target_position` / `path_position` — where this creature will be in
  *t* seconds, used by anyone aiming at it or planning around it.
- `accessible` — restrictor query, forwarded.
- `extrapolate_path` — whether the detail path may continue past the last level vertex.
- `set_build_path_at_once` — forbid deferring path work to a worker.
- `teleport` — the cross-level transition hook, overridden per creature kind.
- `on_restrictions_change` / `on_travel_point_change` / `on_build_path` — notification hooks.
- `can_use_distributed_computations` — whether this stage may be handed to a worker.
- `create_restricted_object` — factory hook so a creature kind can supply its own
  restrictor semantics.
- `build_level_path` — run the level search inline instead of deferring it.
