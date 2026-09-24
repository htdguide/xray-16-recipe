# src/xrGame/cover_evaluators.h

> Declares the six cover evaluators implemented in [`cover_evaluators.cpp`](cover_evaluators.cpp.md).

**Needs** — [`cover_evaluators_inline.h`](cover_evaluators_inline.h.md) · [`cover_point.h`](cover_point.h.md) · [`restricted_object.h`](restricted_object.h.md) · [`xrAICore/Navigation/game_graph_space.h`](../xrAICore/Navigation/game_graph_space.h.md)
**Used by** — [`base_monster_path.cpp`](ai/monsters/basemonster/base_monster_path.cpp.md) · [`base_monster_startup.cpp`](ai/monsters/basemonster/base_monster_startup.cpp.md) · [`control_path_builder_base.cpp`](ai/monsters/control_path_builder_base.cpp.md) · [`control_path_builder_base.h`](ai/monsters/control_path_builder_base.h.md) · [`control_path_builder_base_path.cpp`](ai/monsters/control_path_builder_base_path.cpp.md) · [`corpse_cover.cpp`](ai/monsters/corpse_cover.cpp.md) · [`corpse_cover.h`](ai/monsters/corpse_cover.h.md) · [`monster_cover_manager.cpp`](ai/monsters/monster_cover_manager.cpp.md) · [`ai_stalker.cpp`](ai/stalker/ai_stalker.cpp.md) · [`ai_stalker_cover.cpp`](ai/stalker/ai_stalker_cover.cpp.md) · [`cover_evaluators.cpp`](cover_evaluators.cpp.md) · [`cover_evaluators_inline.h`](cover_evaluators_inline.h.md) · [`stalker_combat_actions.cpp`](stalker_combat_actions.cpp.md) · [`stalker_danger_in_direction_actions.cpp`](stalker_danger_in_direction_actions.cpp.md) · _and 3 more_
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in
[`cover_evaluators.cpp`](cover_evaluators.cpp.md), with the setup and lifecycle operations
split into [`cover_evaluators_inline.h`](cover_evaluators_inline.h.md). The hierarchy is
shallow and its shape carries one decision: the distance-band tests live in the
close-to-enemy evaluator, and three others derive from it purely to inherit them.

Exported units:

- **`CCoverEvaluatorBase`** — the protocol: `setup`, `initialize`, `evaluate` per candidate,
  `finalize`; `selected` and `loophole` for the answer; `inertia`, `actual` and `invalidate`
  for reuse; `accessible` for the restrictor test; the two smart-cover permissions. Two
  evaluation methods are required of every implementor, one for plain cover points and one
  for smart covers.
- **`CCoverEvaluatorCloseToEnemy`** — the distance band plus "get closer".
- **`CCoverEvaluatorFarFromEnemy`** — the same band plus "get further".
- **`CCoverEvaluatorBest`** — the combat evaluator: directional cover quality, a
  threat-on-the-way rejection, and smart-cover firing positions.
- **`CCoverEvaluatorAngle`** — alignment with the most open direction at a reference vertex.
- **`CCoverEvaluatorSafe`** — omnidirectional concealment, no threat needed.
- **`CCoverEvaluatorAmbush`** — concealed from the enemy, exposed to a watched point.
