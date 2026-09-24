# src/xrGame/stalker_combat_action_base.h

> Declares the base shared by every combat action: the firing helpers, the burst-parameter selector and the combat voice lines.

**Needs** — [`stalker_combat_action_base.cpp`](stalker_combat_action_base.cpp.md) · [`stalker_base_action.h`](stalker_base_action.h.md) · [`cover_point.h`](cover_point.h.md)
**Used by** — [`smart_cover_planner_actions.h`](smart_cover_planner_actions.h.md) · [`stalker_combat_action_base.cpp`](stalker_combat_action_base.cpp.md) · [`stalker_combat_actions.h`](stalker_combat_actions.h.md) · [`stalker_low_cover_actions.h`](stalker_low_cover_actions.h.md) · [`stalker_search_actions.cpp`](stalker_search_actions.cpp.md) · [`stalker_search_actions.h`](stalker_search_actions.h.md)
**Tier floor** — T2: per-action objects calling into the creature's weapon and sound managers.

## Purpose

Declares the surface implemented in
[`stalker_combat_action_base.cpp`](stalker_combat_action_base.cpp.md). It sits between the
generic action base and the two dozen concrete combat actions, and it exists because those
actions all need the same four things — point the weapon, decide the burst shape, say
something, and move to a cover point — and none of them should own the answer.

## Exported units

- `initialize()` / `finalize()` — combat entry and exit; manage the creature's sound mask.
- `setup_cover(cover)` — send the creature to a cover point, smart or ordinary.
- `select_queue_params(distance, ...)` — choose burst length and inter-burst pause from the
  weapon class and the range to the target.
- `fire_make_sense()` — may this creature usefully pull the trigger at all.
- `fire()` — fire a burst, or keep turning if not yet aimed.
- `aim_ready()` / `aim_ready_force_full()` — bring the weapon to the aimed posture; the
  second refuses the shortened aim variant.
- `play_panic_sound` / `play_attack_sound` / `play_start_search_sound` /
  `play_enemy_lost_sound` — the four combat voice lines, each with a start/stop time window
  and an optional identifier for deduplication.
