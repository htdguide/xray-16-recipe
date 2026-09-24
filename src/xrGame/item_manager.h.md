# src/xrGame/item_manager.h

> Declares a creature's memory of the pickable items it has seen: which ones are worth going for, and which one is currently chosen.

**Needs** — [`object_manager.h`](object_manager.h.md) · [`GameObject.h`](GameObject.h.md)
**Used by** — [`CustomMonster.cpp`](CustomMonster.cpp.md) · [`agent_manager_properties.cpp`](agent_manager_properties.cpp.md) · [`monster_corpse_memory.cpp`](ai/monsters/monster_corpse_memory.cpp.md) · [`ai_rat_fire.cpp`](ai/monsters/rats/ai_rat_fire.cpp.md) · [`rat_state_activation.cpp`](ai/monsters/rats/rat_state_activation.cpp.md) · [`rat_state_switch.cpp`](ai/monsters/rats/rat_state_switch.cpp.md) · [`ai_stalker_misc.cpp`](ai/stalker/ai_stalker_misc.cpp.md) · [`item_manager.cpp`](item_manager.cpp.md) · [`item_manager_inline.h`](item_manager_inline.h.md) · [`memory_manager.cpp`](memory_manager.cpp.md) · [`memory_manager.h`](memory_manager.h.md) · [`script_game_object2.cpp`](script_game_object2.cpp.md) · [`stalker_alife_actions.cpp`](stalker_alife_actions.cpp.md) · [`stalker_property_evaluators.cpp`](stalker_property_evaluators.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CItemManager`, one specialization of the generic "remember a set of objects and
pick the best one" machinery in [`object_manager.h`](object_manager.h.md) — the same
machinery that backs enemy and danger selection. See
[`item_manager.cpp`](item_manager.cpp.md) for the substance.

Exported units:

- `CItemManager` — holds the owning creature and, when that creature is a stalker, a
  second reference to it in its stalker form.
- `useful` — the admission filter: whether an item may enter the remembered set at all.
- `is_useful` — the same question deferred to the creature, so a script or a subclass can
  override it.
- `evaluate` · `do_evaluate` — the ranking function and the creature's override hook.
- `update` — re-runs selection for this frame.
- `remove_links` — forgets a destroyed object.
- `on_restrictions_change` — drops the selection when it becomes unreachable.
