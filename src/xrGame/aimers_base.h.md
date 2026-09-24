# src/xrGame/aimers_base.h

> Declares the shared base of every aimer: the aiming solver, the bone-sampling helper and the skeleton hook.

**Needs** — [`aimers_base.cpp`](aimers_base.cpp.md) · [`aimers_base_inline.h`](aimers_base_inline.h.md) · [`animation_movement_controller.h`](animation_movement_controller.h.md) · [`Include/xrRender/animation_motion.h`](../Include/xrRender/animation_motion.h.md)
**Used by** — [`aimers_base.cpp`](aimers_base.cpp.md) · [`aimers_base_inline.h`](aimers_base_inline.h.md) · [`aimers_bone.h`](aimers_bone.h.md) · [`aimers_bone_inline.h`](aimers_bone_inline.h.md) · [`aimers_weapon.cpp`](aimers_weapon.cpp.md) · [`aimers_weapon.h`](aimers_weapon.h.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the surface implemented in [`aimers_base.cpp`](aimers_base.cpp.md) and
[`aimers_base_inline.h`](aimers_base_inline.h.md), and holds the state every aimer shares
(the object, its skeleton and animation interfaces, the target, the motion being sampled
and the reference transform).

Exported units, all for derived aimers only:

- **construction** from an object, a motion name, a first-or-last-frame flag and a target;
- **`aim_at_position`** — the solver; see the cpp twin;
- **`aim_at_direction`** — declared and never defined, so the *aim along a direction rather
  than at a point* variant does not exist; a rebuild should drop it or write it;
- **`fill_bones`** — sample a set of bones' poses under the motion without disturbing the
  visible animation; see the inline twin;
- **`callback`** — the skeleton hook that applies a correction to one bone.

Aimers are non-copyable, which states what they are: a short-lived computation holding
references into an object and a target the caller owns.
