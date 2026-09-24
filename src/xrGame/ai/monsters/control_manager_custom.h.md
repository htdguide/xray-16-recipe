# src/xrGame/ai/monsters/control_manager_custom.h

> Declares the creature's ability roster, implemented in [`control_manager_custom.cpp`](control_manager_custom.cpp.md).

**Needs** — [`control_combase.h`](control_combase.h.md) · [`anim_triple.h`](anim_triple.h.md) · [`control_jump.h`](control_jump.h.md) · [`control_rotation_jump.h`](control_rotation_jump.h.md) · [`control_melee_jump.h`](control_melee_jump.h.md)
**Used by** — [`base_monster.h`](basemonster/base_monster.h.md) · [`control_manager_custom.cpp`](control_manager_custom.cpp.md) · [`controller_state_control_hit_inline.h`](controller/controller_state_control_hit_inline.h.md) · [`dog.cpp`](dog/dog.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CControlManagerCustom`, the base element that owns whichever abilities a creature
was granted. Substance is in
[`control_manager_custom.cpp`](control_manager_custom.cpp.md).

## State

One optional handle per ability — clip sequencer, animation triad, rotation jump, jump,
run-through attack, threat display, melee jump, critical wound — plus the authored data a
creature supplies for the three abilities whose payload is fixed at load: a list of rotation
variants, one melee-jump pair, and the threat clip and its timing.

Exported units, grouped as the header groups them:

- **lifecycle** — `reinit`, `update_frame` (empty), `update_schedule` (the ability poll),
  `on_start_control`, `on_stop_control`, `on_event`.
- **roster** — `add_ability`, one call per ability the creature wants.
- **sequencer** — `seq_init`, `seq_add`, `seq_switch`, `seq_run`.
- **animation triad** — `ta_fill_data`, `ta_activate`, `ta_pointbreak`, `ta_is_active`
  (with and without a triad to compare against), `ta_deactivate`.
- **jump** — `jump` in three spellings, `script_jump`, `load_jump_data`, `is_jumping`,
  `check_if_jump_possible`, `jump_if_possible`, `get_jump_control`.
- **scripted channel control** — `script_capture`, `script_release`.
- **authoring** — `add_rotation_jump_data`, `add_melee_jump_data`, `set_threaten_data`,
  `critical_wound`.
- **housekeeping** — `remove_links`.
- **private polls** — `check_attack_jump`, `check_jump_over_physics` (dead),
  `check_rotation_jump`, `check_melee_jump`, `check_run_attack`, `check_threaten`, and
  `fill_rotation_data`.
