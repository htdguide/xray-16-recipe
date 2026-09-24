# src/xrGame/enemy_manager.h

> Declares the per-creature enemy selection, implemented in [`enemy_manager.cpp`](enemy_manager.cpp.md) and [`enemy_manager_inline.h`](enemy_manager_inline.h.md).

**Needs** — [`object_manager.h`](object_manager.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`CustomMonster.h`](CustomMonster.h.md) · [`enemy_manager_inline.h`](enemy_manager_inline.h.md) · [`xrScriptEngine/script_callback_ex.h`](../xrScriptEngine/script_callback_ex.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`CustomMonster.cpp`](CustomMonster.cpp.md) · [`agent_enemy_manager.cpp`](agent_enemy_manager.cpp.md) · [`agent_manager_properties.cpp`](agent_manager_properties.cpp.md) · [`ai_monsters_misc.cpp`](ai/ai_monsters_misc.cpp.md) · [`monster_enemy_memory.cpp`](ai/monsters/monster_enemy_memory.cpp.md) · [`ai_rat_feel.cpp`](ai/monsters/rats/ai_rat_feel.cpp.md) · [`rat_state_switch.cpp`](ai/monsters/rats/rat_state_switch.cpp.md) · [`ai_stalker_fire.cpp`](ai/stalker/ai_stalker_fire.cpp.md) · [`ai_stalker_misc.cpp`](ai/stalker/ai_stalker_misc.cpp.md) · [`danger_manager.cpp`](danger_manager.cpp.md) · [`enemy_manager.cpp`](enemy_manager.cpp.md) · [`enemy_manager_inline.h`](enemy_manager_inline.h.md) · [`memory_manager.cpp`](memory_manager.cpp.md) · [`memory_manager.h`](memory_manager.h.md) · _and 24 more_
**Tier floor** — T3: a declaration

## Purpose

Declares the enemy manager: a scored object selection specialised to living entities.
Substance is in [`enemy_manager.cpp`](enemy_manager.cpp.md).

Its own load-bearing content is that this is a **specialisation of the generic scored object
manager** — the list, the selection and the update loop are inherited, and this class
supplies only the filter, the score and the hysteresis. A reader looking for how candidates
are gathered should read [`object_manager.h`](object_manager.h.md) first.

Exported units:

- `CEnemyManager` — the selection.
- `useful` / `is_useful` — the candidacy filter, and the indirection through the creature so
  a subclass can override it.
- `evaluate` / `do_evaluate` — the score, and the same indirection.
- `update` — the per-update entry point, including the autosave gate.
- `selected` / `set_enemy` / `invalidate_enemy` — the current enemy, plus the forced-enemy
  override used by smart covers.
- `last_enemy` / `last_enemy_time` — the enemy just gone, kept after selection clears.
- `enable_enemy_change` — pin the current enemy against loss of sight.
- `wounded` — nominate a wounded enemy.
- `useful_callback` — the script veto on any candidate.
- `ignore_monster_threshold` / `max_ignore_monster_distance` and their restore pairs — the
  "a human may walk past a weak monster" rule, settable from script and restorable from the
  creature's own configuration.
- `remove_links` — clear every reference to a destroyed object.
- `reload` — re-read the tunables and clear the history.
- `set_ready_to_save` — release this creature's autosave hold from outside.
- `enemy_inertia` / `need_update` / `process_wounded` / `change_from_wounded` /
  `remove_wounded` / `try_change_enemy` / `on_enemy_change` / `expedient` — the private and
  protected machinery, each substantive; see the implementation twin.
