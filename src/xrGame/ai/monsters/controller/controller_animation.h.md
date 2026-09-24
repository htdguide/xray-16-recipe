# src/xrGame/ai/monsters/controller/controller_animation.h

> Declares the controller's two-partition animation driver, implemented in [`controller_animation.cpp`](controller_animation.cpp.md).

**Needs** — [`../control_animation_base.h`](../control_animation_base.h.md) · [`../ai_monster_defs.h`](../ai_monster_defs.h.md)
**Used by** — [`controller.cpp`](controller.cpp.md) · [`controller_animation.cpp`](controller_animation.cpp.md) · [`controller_state_attack_camp_inline.h`](controller_state_attack_camp_inline.h.md) · [`controller_state_attack_fire_inline.h`](controller_state_attack_fire_inline.h.md) · [`controller_state_attack_hide_inline.h`](controller_state_attack_hide_inline.h.md) · [`controller_state_attack_hide_lite_inline.h`](controller_state_attack_hide_lite_inline.h.md) · [`controller_state_attack_moveout_inline.h`](controller_state_attack_moveout_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CControllerAnimation`, the creature-specific base driver of the animation channel
that gives this creature a separately chosen torso clip and legs clip. Substance, including
the extent to which the design is disabled in the shipped build, is in
[`controller_animation.cpp`](controller_animation.cpp.md).

## State

Two clip tables, a path-rotation table per moving family, the current legs and torso actions,
and a latch that protects a playing psi-attack torso clip.

The legs-action enumeration is declared here and is a **tagged bit set**: a family bit from
bit sixteen up — standing, sneaking, sneak-moving, walking, running — with a small index in
the low bits. A single mask test answers "is this creature running", which the movement test,
the path-rotation lookup and the standing-pose search all rely on. The torso actions are a
plain four-value enumeration: idle, sneaking, psi attack, running.

Exported units:

- `reinit`, `update_frame`, `on_event`, `on_start_control`, `on_stop_control` — the driver
  lifecycle.
- `load`, `add_path_rotation` — resolve the clips by literal name and register the
  angle-to-clip table for each moving family.
- `set_body_state` — set the torso and legs actions. As shipped it ignores its arguments and
  pins the creature to the sneaking pose.
- `set_path_params` — choose the forward or backward gait from whether the destination is in
  front of where the creature is looking, and enable the path. This one is live and is what
  the creature's states call.
- `on_switch_controller` — the mental-state hook; re-selects both partitions in the danger
  state and the whole body otherwise.
- `select_velocity`, `set_path_direction`, `select_torso_animation`, `select_legs_animation`,
  `get_path_rotation`, `is_moving` — private; the two-partition selection, most of it
  unreachable in the shipped build.
