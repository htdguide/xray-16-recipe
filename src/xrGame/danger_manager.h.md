# src/xrGame/danger_manager.h

> Declares the per-creature danger list implemented in [`danger_manager.cpp`](danger_manager.cpp.md).

**Needs** — [`danger_object.h`](danger_object.h.md) · [`danger_manager_inline.h`](danger_manager_inline.h.md)
**Used by** — [`CustomMonster.cpp`](CustomMonster.cpp.md) · [`agent_manager_properties.cpp`](agent_manager_properties.cpp.md) · [`ai_stalker_misc.cpp`](ai/stalker/ai_stalker_misc.cpp.md) · [`danger_manager.cpp`](danger_manager.cpp.md) · [`danger_manager_inline.h`](danger_manager_inline.h.md) · [`memory_manager.cpp`](memory_manager.cpp.md) · [`memory_manager.h`](memory_manager.h.md) · [`script_game_object2.cpp`](script_game_object2.cpp.md) · [`stalker_combat_actions.cpp`](stalker_combat_actions.cpp.md) · [`stalker_combat_planner.cpp`](stalker_combat_planner.cpp.md) · [`stalker_danger_grenade_actions.cpp`](stalker_danger_grenade_actions.cpp.md) · [`stalker_danger_in_direction_actions.cpp`](stalker_danger_in_direction_actions.cpp.md) · [`stalker_danger_planner.cpp`](stalker_danger_planner.cpp.md) · [`stalker_danger_property_evaluators.cpp`](stalker_danger_property_evaluators.cpp.md) · _and 3 more_
**Tier floor** — T3: a declaration

## Purpose

Declares `CDangerManager`, the threat half of a creature's memory. Substance is in
[`danger_manager.cpp`](danger_manager.cpp.md).

Exported units:

- construction, `reinit`, `reset` — bind the owning creature, and the two clearing depths
  (full versus keep-the-ignore-list).
- `Load` / `reload` — the configuration hooks, both empty.
- `update` — prune, then select the most urgent record.
- `add` — four overloads: from a visibility record, from a sound record, from a hit record,
  and from an already-classified danger record. The first three classify and funnel into the
  fourth.
- `useful` / `is_useful` / `evaluate` / `do_evaluate` — the ignore-and-age test, the
  creature's veto, the creature's score, and the built-in score.
- `remove` / `remove_links` / `ignore` — drop one record, the object-destroyed sweep, and
  the permanent suppression list.
- `time_line` — read and write the cutoff timestamp below which records are discarded.
- `selected` / `objects` — the winner and the whole list.
- `save` / `load` — persist the ignore list.

**Notes** — nearly every method is overridable, because a creature class is expected to
replace the scoring policy. The data members are private and the list is exposed read-only,
so a subclass rescores but does not restructure.
