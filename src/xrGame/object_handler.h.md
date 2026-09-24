# src/xrGame/object_handler.h

> Declares the creature-side facade over the object-handling planner, implemented in [`object_handler.cpp`](object_handler.cpp.md).

**Needs** — [`InventoryOwner.h`](InventoryOwner.h.md) · [`object_handler_inline.h`](object_handler_inline.h.md) · [`xrAICore/Navigation/graph_engine_space.h`](../xrAICore/Navigation/graph_engine_space.h.md)
**Used by** — [`ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`object_actions_inline.h`](object_actions_inline.h.md) · [`object_handler.cpp`](object_handler.cpp.md) · [`object_handler_inline.h`](object_handler_inline.h.md) · [`script_game_object2.cpp`](script_game_object2.cpp.md) · [`stalker_combat_action_base.cpp`](stalker_combat_action_base.cpp.md) · [`stalker_danger_by_sound_actions.cpp`](stalker_danger_by_sound_actions.cpp.md) · [`stalker_danger_grenade_actions.cpp`](stalker_danger_grenade_actions.cpp.md) · [`stalker_danger_in_direction_actions.cpp`](stalker_danger_in_direction_actions.cpp.md) · [`stalker_danger_unknown_actions.cpp`](stalker_danger_unknown_actions.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CObjectHandler`, the layer a stalker mixes in to gain "hands": it owns the
object-handling planner, keeps the planner's item set in step with the inventory, and answers
the questions the rest of the creature asks about what is in those hands. Substance is in
[`object_handler.cpp`](object_handler.cpp.md).

Exported units:

- `CObjectHandler` — extends the inventory owner. Holds the planner, the three hand bones, a
  cached pair of sling bones, the infinite-ammunition flag and the clutched-hammer state.
- `reinit` — rebind to a creature and resolve its bone names.
- `net_Spawn` — take the infinite-ammunition flag from the server record.
- `update` — step the planner, once per creature update.
- `OnItemTake` / `OnItemDrop` / `attach` / `detach` — keep the planner's item set, the torch
  and the recoil effector in step with inventory changes.
- `set_goal` (two forms) — order the planner to reach a named object action with a named
  object, with burst-size and burst-interval bounds.
- `goal_reached` — whether the plan has run out.
- `best_weapon` — the weapon the creature's own evaluation prefers against its enemy.
- `weapon_bones` — which three bones the held weapon attaches to, which differs between held
  and slung.
- `weapon_strapped` / `weapon_unstrapped` / `is_weapon_going_to_be_strapped` — the three
  questions about sling state, each with a different answer mid-transition.
- `aim_time` (set and get) — how long the creature is made to hold an aim.
- `hammer_is_clutched` / `infinite_ammo` / `planner` — state readers.
- `can_use_dynamic_lights` — whether a creature's torch casts real light.
