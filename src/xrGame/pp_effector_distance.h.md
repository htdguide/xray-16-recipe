# src/xrGame/pp_effector_distance.h

> Declares the distance-driven post-process effector controller.

**Needs** — [`pp_effector_custom.h`](pp_effector_custom.h.md)
**Used by** — [`pp_effector_distance.cpp`](pp_effector_distance.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in
[`pp_effector_distance.cpp`](pp_effector_distance.cpp.md): a controller that turns a
viewer-to-source distance into a post-process blend factor.

Exported units:

- `load` — read the inner and outer radius fractions from a configuration section.
- `set_radius` · `set_current_dist` — the two values the owner pushes in each frame.
- `check_start_conditions` · `check_completion` — the activation band.
- `update_factor` — the ramp.
- `create_effector` — factory for the effector this controller drives.
