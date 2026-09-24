# src/xrGame/ai/monsters/control_manager.h

> Declares the per-creature control bus implemented in [`control_manager.cpp`](control_manager.cpp.md).

**Needs** — [`control_com_defs.h`](control_com_defs.h.md) · [`control_animation.h`](control_animation.h.md) · [`control_direction.h`](control_direction.h.md) · [`control_path_builder.h`](control_path_builder.h.md) · [`control_movement.h`](control_movement.h.md)
**Used by** — [`anim_triple.cpp`](anim_triple.cpp.md) · [`base_monster.cpp`](basemonster/base_monster.cpp.md) · [`base_monster.h`](basemonster/base_monster.h.md) · [`base_monster_debug.cpp`](basemonster/base_monster_debug.cpp.md) · [`control_animation.cpp`](control_animation.cpp.md) · [`control_combase.h`](control_combase.h.md) · [`control_direction.cpp`](control_direction.cpp.md) · [`control_jump.cpp`](control_jump.cpp.md) · [`control_manager.cpp`](control_manager.cpp.md) · [`control_manager_custom.cpp`](control_manager_custom.cpp.md) · [`control_melee_jump.cpp`](control_melee_jump.cpp.md) · [`control_movement.cpp`](control_movement.cpp.md) · [`control_path_builder.cpp`](control_path_builder.cpp.md) · [`control_rotation_jump.cpp`](control_rotation_jump.cpp.md) · _and 1 more_
**Tier floor** — T3: a declaration

## Purpose

Declares `CControl_Manager`, the object every creature ability in chapter 24 is written
against. Substance is in [`control_manager.cpp`](control_manager.cpp.md).

## State

Declared here, described in the implementation twin.

Exported units, grouped as the header groups them:

- **lifecycle** — `init_external`, `load`, `reinit`, `reload`, `update_schedule`,
  `update_frame`.
- **events** — `notify`, `subscribe`, `unsubscribe`.
- **registration** — `add`, `set_base_controller`, `install_path_manager`.
- **arbitration** — `capture`, `release`, `capture_pure`, `release_pure`,
  `check_capturer`, `get_capturer`, `com_type`, `is_captured`, `is_captured_pure`,
  `lock`, `unlock`, `activate`, `deactivate`, `check_start_conditions`.
- **payload access** — `data`.
- **direct accessors** — `animation`, `direction`, `path_builder`, `movement`, each
  returning the channel's element by reference. These are how the rest of the chapter
  reaches the body: an ability says "the creature's direction" and gets the live element,
  not a copy.
- **conveniences** — `path_stop`, `move_stop`, `dir_stop`, `build_path_line`.
- **diagnostics** — a debug-only dump of every element's channel, activity, capturer and
  lock state into a tree the engine's on-screen inspector renders. It is the intended way
  to debug a creature that has stopped moving, and a rebuild that omits it will regret it.

The private helpers `is_pure`, `is_base`, `is_locked` and `check_active_com` encode the
element-shape inference and the active-list bookkeeping described in the implementation
twin.
