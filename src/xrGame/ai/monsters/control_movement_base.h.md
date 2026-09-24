# src/xrGame/ai/monsters/control_movement_base.h

> Declares the base driver of the movement channel and the owner of the creature's authored speed table, implemented in [`control_movement_base.cpp`](control_movement_base.cpp.md).

**Needs** — [`control_combase.h`](control_combase.h.md) · [`ai_monster_defs.h`](ai_monster_defs.h.md)
**Used by** — [`base_monster.cpp`](basemonster/base_monster.cpp.md) · [`base_monster_startup.cpp`](basemonster/base_monster_startup.cpp.md) · [`bloodsucker.cpp`](bloodsucker/bloodsucker.cpp.md) · [`boar.cpp`](boar/boar.cpp.md) · [`burer.cpp`](burer/burer.cpp.md) · [`cat.cpp`](cat/cat.cpp.md) · [`chimera.cpp`](chimera/chimera.cpp.md) · [`control_animation_base.cpp`](control_animation_base.cpp.md) · [`control_animation_base_update.cpp`](control_animation_base_update.cpp.md) · [`control_critical_wound.cpp`](control_critical_wound.cpp.md) · [`control_jump.cpp`](control_jump.cpp.md) · [`control_movement_base.cpp`](control_movement_base.cpp.md) · [`control_rotation_jump.cpp`](control_rotation_jump.cpp.md) · [`control_run_attack.cpp`](control_run_attack.cpp.md) · _and 9 more_
**Tier floor** — T3: a declaration

## Purpose

Declares `CControlMovementBase`. Substance is in
[`control_movement_base.cpp`](control_movement_base.cpp.md).

## State

A map from gait identifier to the authored speed record, plus the currently commanded
speed and acceleration.

Exported units:

- `load`, `load_velocity` — read the ten named gaits from the creature's configuration
  section and register each with the detailed path manager.
- `get_velocity` — look one up; a missing gait is a hard failure.
- `reinit`, `update_frame` — reset and capture the channel; publish the held values.
- `set_velocity`, `set_accel`, `stop`, `stop_accel` — command a speed, with the
  acceleration chosen from the creature's authored accelerate and brake rates according to
  which way the speed is changing.
- `get_velocity_from_path` — the speed the current waypoint calls for, looking past a
  waypoint that merely marks a corner.
