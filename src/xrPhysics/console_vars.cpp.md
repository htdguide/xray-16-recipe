# src/xrPhysics/console_vars.cpp

> The defaults for the physics module's six run-time tunables, and what each one buys.

**Needs** — [`console_vars.h`](console_vars.h.md) · [`PhysicsCommon.h`](PhysicsCommon.h.md)
**Used by** — [`console_vars.h`](console_vars.h.md)
**Tier floor** — T3: scalar defaults.

## Purpose

Defines the six values declared in [`console_vars.h`](console_vars.h.md). Each default is a
measured trade rather than a round number, and the reasons are the content of this page.

## State

```text
debug_dump_physics_step   : bool  = false
tri_query_ex_aabb_rate    : real  = 1.3
tri_clear_disable_count   : int   = 10
break_common_factor       : real  = 0.01
rigid_break_weapon_factor : real  = 1.0
step_time                 : real  = fixed_step     # the solver's timestep, 0.01 s
```

## `tri_query_ex_aabb_rate`

**Contract** — the factor by which a moving geometry's own bounding box is enlarged before it is used
to query the static collision database for triangles.

**Notes** — this is the *hysteresis* on triangle caching, and it is the single most performance-
sensitive number in the collider. The cached triangle list for a geometry is reused for as long as
the geometry's current box stays inside the box that was queried, so the surplus 30% buys roughly
that much travel before another tree query is needed
(see [`dSortTriPrimitive.h`](tri-colliderknoopc/dSortTriPrimitive.h.md)). Raising it cuts queries
and grows the per-step triangle loop; lowering it does the reverse. 1.3 is where the two costs
balanced on hardware of the era; a rebuild should re-measure rather than inherit it.

## `tri_clear_disable_count`

**Contract** — the number of consecutive steps an object must remain asleep before its cached
triangle list is released.

**Notes** — the delay exists because sleeping is not a commitment: an object that settles and is
immediately nudged would otherwise pay a fresh tree query. Ten steps is a tenth of a second at the
solver's rate — long enough to cover a bounce, short enough that a room full of settled debris does
not hold thousands of triangle indices.

## `break_common_factor` and `rigid_break_weapon_factor`

**Contract** — the global scale from impact energy to accumulated breakage on a breakable joint, and
an additional multiplier applied when the impulse came from a weapon rather than from a collision.

**Notes** — the two exist separately because breakables must survive being walked into and shot off
their mounts, and those are different energy regimes. The common factor of 0.01 sets the scale
against which the per-joint breakage limits in the shipped model data were authored; changing it
re-tunes every breakable in the game at once, which is exactly what it is for and exactly why it is
not something a rebuild may pick freely — the limits are in the data.

## `step_time`

**Contract** — the solver timestep, initialized from the module's fixed step.

**Notes** — exposing it as a console value is a debugging affordance, not a licence to vary it.
The determinism requirement means the step must be constant within a session; the outer loop varies
the *number* of steps, never their length (see [`PHWorld.cpp`](PHWorld.cpp.md)).
