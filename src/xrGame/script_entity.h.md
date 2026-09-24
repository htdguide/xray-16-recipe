# src/xrGame/script_entity.h

> Declares the script-driven action queue mixin.

**Needs** — [`script_entity.cpp`](script_entity.cpp.md) · [`script_entity_space.h`](script_entity_space.h.md) · [`script_entity_inline.h`](script_entity_inline.h.md) · [`script_entity_action.h`](script_entity_action.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md)
**Used by** — [`Car.cpp`](Car.cpp.md) · [`Car.h`](Car.h.md) · [`CustomMonster.cpp`](CustomMonster.cpp.md) · [`CustomMonster.h`](CustomMonster.h.md) · [`ai_trader.h`](ai/trader/ai_trader.h.md) · [`script_entity.cpp`](script_entity.cpp.md) · [`script_entity_inline.h`](script_entity_inline.h.md) · [`script_game_object.cpp`](script_game_object.cpp.md) · [`script_object.cpp`](script_object.cpp.md) · [`script_object.h`](script_object.h.md)
**Tier floor** — T2: a declaration

## Purpose

Declares the surface implemented in [`script_entity.cpp`](script_entity.cpp.md). It is a
*mixin*: concrete entity classes inherit it next to their own behaviour and chain its
lifecycle hooks from theirs. The header therefore also fixes which of its methods are
overridable by those concrete classes — chiefly the six `assign_*` channel projections,
which creature types with unusual movement or sound replace.

Exported units:

- `SavedSound` — a deferred heard-sound event: emitter identity, AI sound type, position,
  power.
- The entity lifecycle: `init`, `reinit`, `net_spawn`, `net_destroy`, `shedule_update`,
  `update_client`, construction.
- Control: `set_script_control`, `get_script_control`, `get_script_control_name`,
  `set_script_capture`, `can_script_capture`.
- Queue: `add_action`, `current_action`, `action_count`, `action_by_index`,
  `clear_action_queue`, `reset_script_data`, `process`, `finish_action`.
- Channel projection: `assign_movement`, `assign_watch`, `assign_animation`,
  `assign_sound`, `assign_particles`, `assign_object`, `assign_monster_action`.
- Per-frame attachment support: `updated_matrix`, `update_sounds`, `update_particles`,
  `start_script_animation`.
- Senses: `sound_callback`, `process_sound_callbacks`, `check_object_visibility`,
  `check_type_visibility`.
- Queries a concrete type overrides: `current_enemy`, `current_corpse`, `enemy_strength`,
  `patrol_path_name`, `check_if_completed`.
- `cast_script_entity` — the downcast the script-visible facade uses to find this mixin on
  an arbitrary entity.
