# src/xrPhysics/console_vars.h

> The physics tuning values the console can change while the game runs.

**Needs** — [`console_vars.cpp`](console_vars.cpp.md) · [`xrPhysics.h`](xrPhysics.h.md)
**Used by** — [`console_commands.cpp`](../xrGame/console_commands.cpp.md) · [`PHObject.cpp`](PHObject.cpp.md) · [`PHWorld.cpp`](PHWorld.cpp.md) · [`console_vars.cpp`](console_vars.cpp.md) · [`dSortTriPrimitive.h`](tri-colliderknoopc/dSortTriPrimitive.h.md)
**Tier floor** — T3: a named group of mutable scalars.

## Purpose

Declares the surface implemented in [`console_vars.cpp`](console_vars.cpp.md), where each value's
default and meaning are given. The values are grouped into one named block rather than scattered as
free variables so that the console registration in [`xrEngine`](../xrEngine/README.md) has a single
place to bind against and so that a reader can see the whole tunable surface of the physics module
at once — it is six numbers, which is the point worth noticing.

- `debug_dump_physics_step` — whether each step's body state is logged.
- `tri_query_ex_aabb_rate` — how much the triangle query box is inflated over the geometry's own.
- `tri_clear_disable_count` — how many steps an object must be asleep before its cached triangles are dropped.
- `break_common_factor` — global scale on breakage from impacts.
- `rigid_break_weapon_factor` — additional scale when the impact came from a weapon.
- `step_time` — the fixed solver timestep.
