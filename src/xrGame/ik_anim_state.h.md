# src/xrGame/ik_anim_state.h

> Declares the four booleans that tell the foot-placement solver what the animation is currently doing with this limb.

**Needs** — [`ik_anim_state.cpp`](ik_anim_state.cpp.md)
**Used by** — [`IKLimbsController.cpp`](IKLimbsController.cpp.md) · [`IKLimb.cpp`](ik/IKLimb.cpp.md) · [`IKLimb.h`](ik/IKLimb.h.md) · [`ik_anim_state.cpp`](ik_anim_state.cpp.md)
**Tier floor** — T2: a declaration

## Purpose

Declares the surface implemented in [`ik_anim_state.cpp`](ik_anim_state.cpp.md). Foot
placement has to know four things about the playing animation before it can decide where to
put a foot, and this small record is the answer to all four, recomputed once per frame per
limb.

Exported units:

- `ik_anim_state` — the four flags plus the blend it last sampled.
- `update` — recompute them from the currently playing blend for one limb.
- `step`, `glue`, `idle`, `blending` — the four answers.
- `auto_unstuck` — always true in the shipped build; the disabled condition is preserved in
  the source as a comment and is not part of the behaviour.
- `time_step_begin` — how long until this limb's next footfall marker.
