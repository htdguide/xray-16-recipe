# src/xrGame/ai/monsters/control_jump.h

> Declares the jump ability and its payload, implemented in [`control_jump.cpp`](control_jump.cpp.md).

**Needs** — [`control_combase.h`](control_combase.h.md)
**Used by** — [`chimera_attack_state_inline.h`](chimera/chimera_attack_state_inline.h.md) · [`control_animation_base.cpp`](control_animation_base.cpp.md) · [`control_jump.cpp`](control_jump.cpp.md) · [`control_manager_custom.cpp`](control_manager_custom.cpp.md) · [`control_manager_custom.h`](control_manager_custom.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CControlJump`, the custom element on the jump channel, and the record a caller
fills in to describe one jump. Substance is in [`control_jump.cpp`](control_jump.cpp.md).

## State

`SControlJumpData` — the channel payload: a target object and position, a force factor
overriding the creature's authored jump factor, a flag set, and a clip (plus gait, where it
matters) for each of the four stages. The flag set is the ability's vocabulary and is
tabulated in the implementation twin.

The four-stage enumeration — prepare, prepare-in-move, glide, ground — is declared here and
its *order* is load-bearing: stage advance is an increment over it.

Exported units:

- `load`, `reinit` — read the nine authored parameters; clear the timers.
- `check_start_conditions`, `activate`, `on_release`, `on_event`, `update_frame` — the
  ability lifecycle.
- `can_jump` (by object, and by position with an aggressive flag) — the distance, angle,
  height, cooldown and stageability tests.
- `jump_intersect_geometry` — would the arc pass through geometry.
- `stop` — raise the jump-end event.
- `relative_time`, `in_auto_aim`, `get_auto_aim_factor`, `get_jump_start_pos`,
  `get_max_distance`, `get_min_distance` — queries a creature uses during or about a jump.
- `setup_data` — direct write access to the payload, used by the creature's custom manager
  to stage a jump before capturing the channel.
- `remove_links` — drop the target object when it is destroyed.
