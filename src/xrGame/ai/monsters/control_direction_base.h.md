# src/xrGame/ai/monsters/control_direction_base.h

> Declares the base driver of the direction channel, implemented in [`control_direction_base.cpp`](control_direction_base.cpp.md).

**Needs** — [`control_combase.h`](control_combase.h.md)
**Used by** — [`base_monster.cpp`](basemonster/base_monster.cpp.md) · [`base_monster_path.cpp`](basemonster/base_monster_path.cpp.md) · [`bloodsucker_vampire_execute_inline.h`](bloodsucker/bloodsucker_vampire_execute_inline.h.md) · [`burer.cpp`](burer/burer.cpp.md) · [`burer_fast_gravi.cpp`](burer/burer_fast_gravi.cpp.md) · [`burer_state_attack_gravi_inline.h`](burer/burer_state_attack_gravi_inline.h.md) · [`burer_state_attack_run_around_inline.h`](burer/burer_state_attack_run_around_inline.h.md) · [`chimera_attack_state_inline.h`](chimera/chimera_attack_state_inline.h.md) · [`chimera_state_threaten_roar_inline.h`](chimera/chimera_state_threaten_roar_inline.h.md) · [`control_animation_base.cpp`](control_animation_base.cpp.md) · [`control_animation_base_update.cpp`](control_animation_base_update.cpp.md) · [`control_critical_wound.cpp`](control_critical_wound.cpp.md) · [`control_direction_base.cpp`](control_direction_base.cpp.md) · [`control_jump.cpp`](control_jump.cpp.md) · _and 8 more_
**Tier floor** — T3: a declaration

## Purpose

Declares `CControlDirectionBase`. Substance is in
[`control_direction_base.cpp`](control_direction_base.cpp.md).

## State

Two axis records — each a target angle, a target speed and an acceleration — plus a
request rate-limit and its timestamp. Only the heading axis is live.

Exported units:

- `reinit`, `update_frame` — reset and capture the channel; publish the held values.
- `use_path_direction` — face along the path, optionally reversed.
- `face_target` (by position and by object) — aim at a thing, rate-limited, with an
  optional offset to the outside of the turn.
- `set_heading`, `set_heading_speed`, `set_delay` — set the target angle, the turn rate
  and the rate limit outright.
- `heading` — read back the whole heading axis record. Creature-specific direction drivers
  derive from this class and extend it; see
  [`controller/controller_direction.h`](controller/controller_direction.h.md) for the one
  in this slice.
