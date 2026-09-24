# src/xrGame/animation_movement_controller.h

> Declares the root-motion controller implemented in [`animation_movement_controller.cpp`](animation_movement_controller.cpp.md).

**Needs** — [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md) · [`poses_blending.h`](poses_blending.h.md)
**Used by** — [`CharacterPhysicsSupport.cpp`](CharacterPhysicsSupport.cpp.md) · [`GameObject.cpp`](GameObject.cpp.md) · [`aimers_base.cpp`](aimers_base.cpp.md) · [`aimers_base.h`](aimers_base.h.md) · [`aimers_weapon.cpp`](aimers_weapon.cpp.md) · [`animation_movement_controller.cpp`](animation_movement_controller.cpp.md) · [`smart_cover_planner_actions.cpp`](smart_cover_planner_actions.cpp.md) · [`stalker_animation_pair.cpp`](stalker_animation_pair.cpp.md)
**Tier floor** — T2: a declaration

## Purpose

Declares the surface implemented in
[`animation_movement_controller.cpp`](animation_movement_controller.cpp.md). The class is
non-copyable and listens for blend destruction — both are lifetime facts the implementation
twin explains.

Exported units:

- **construction / destruction** — install and remove animation control over an object's
  world transform.
- **`OnFrame`** — advance the object along the animation's root motion.
- **`NewBlend`** — hand control to the next motion in a chain.
- **`IsActive`** — whether a controlling blend is present.
- **`IsBlending`** — whether that blend is still accruing weight.
- **`stop`** — break the chain, so the next blend restarts rather than accumulates.
- **`ObjStartXform`, `start_transform`, `ControlBlend`** — readers of the base transform
  and the controlling blend.
- **`DBG_verify_position_not_chaged`** — the debug-only assertion that nobody else is
  writing the controlled transform.
