# src/xrGame/ai/monsters/controller/controller_direction.h

> Declares the controller's head-and-spine aiming driver, implemented in [`controller_direction.cpp`](controller_direction.cpp.md).

**Needs** — [`../control_direction_base.h`](../control_direction_base.h.md) · [`../ai_monster_bones.h`](../ai_monster_bones.h.md) · [`../ai_monster_space.h`](../../../ai_monster_space.h.md)
**Used by** — [`controller.cpp`](controller.cpp.md) · [`controller_animation.cpp`](controller_animation.cpp.md) · [`controller_direction.cpp`](controller_direction.cpp.md) · [`controller_state_attack_camp_inline.h`](controller_state_attack_camp_inline.h.md) · [`controller_state_attack_fire_inline.h`](controller_state_attack_fire_inline.h.md) · [`controller_state_attack_hide_inline.h`](controller_state_attack_hide_inline.h.md) · [`controller_state_attack_moveout_inline.h`](controller_state_attack_moveout_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CControllerDirection`, the creature-specific base driver of the direction channel
that gives this creature a gaze independent of its facing. Substance is in
[`controller_direction.cpp`](controller_direction.cpp.md).

## State

The bone manipulator, handles on the spine and head bones, the published gaze orientation,
and the point currently being looked at. Four tuning constants — the two bone limits, a
rotation-speed scale and a floor — live in the implementation file rather than in data.

Exported units:

- `reinit`, `update_schedule` — reset and recompute the gaze each scheduled tick.
- `head_look_point` — aim the gaze at a world point, splitting the rotation between the two
  bones in proportion to their limits.
- `get_head_look_point` — read back what is being looked at. The creature's animation driver
  uses it to decide which directional legs clip to play.
- `get_head_orientation` — the published gaze. This is what the creature's psi-attack
  conditions measure against, and what the base monster's head-orientation query returns for
  this creature.
- `bone_callback`, `assign_bones`, `update_head_orientation` — private; install and drive the
  per-bone rotation.
