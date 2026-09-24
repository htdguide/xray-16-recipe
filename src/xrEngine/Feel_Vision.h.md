# src/xrEngine/Feel_Vision.h

> Declares the sight sense: a potentially-visible set, per-target fuzzy visibility that builds and decays over time, and the ray-cache that makes it affordable.

**Needs** — [`xrCDB/xr_collide_defs.h`](../xrCDB/xr_collide_defs.h.md) · [`Render.h`](Render.h.md) · [`pure_relcase.h`](pure_relcase.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`Feel_Vision.cpp`](Feel_Vision.cpp.md) · [`CustomMonster.h`](../xrGame/CustomMonster.h.md) · [`vision_client.h`](../xrGame/vision_client.h.md)
**Tier floor** — T2.

## Purpose

Declares the surface implemented in [`Feel_Vision.cpp`](Feel_Vision.cpp.md), and the four tuning constants that define what "seeing" means.

## State

```text
# The four constants that define the sense's temporal behaviour.
fuzzy_gain_per_second = 1000    # how fast confidence builds while a target is traced visible
fuzzy_loss_per_second = 1000    # how fast it decays while traced invisible
guaranteed_distance   = 0.001   # closer than this, a target is visible without a trace
position_granularity  = 0.1     # positions within this are treated as unchanged

RECORD VisibleItem
  target           : entity
  fuzzy            : real in [-1, 1]   # < 0 not seen, > 0 seen; magnitude is confidence
  aim_point_local  : vector3           # a point on the target's mesh, in the target's frame
  aim_bone         : int
  last_visible     : vector3           # the last world point found visible on this target
  trace_from       : vector3           # the viewpoint and target position of the last trace,
  trace_to         : vector3           #   so a repeated trace can be skipped
  ray_cache        : the triangle that last blocked this ray, plus the ray it blocked
  cached_visibility: real
```

Visibility is not a boolean. Each candidate carries a signed confidence that rises while traces succeed and falls while they fail, and "visible" means the confidence is positive. That is what makes a creature take a moment to notice something and a moment to lose it, and it is why a target flickering behind a fence does not make the AI flicker with it.

The gain and loss rates are both a thousand per second, which with the clamp below means the confidence saturates in about a millisecond of accumulated sense time. That looks instantaneous, and it is: the delay a player perceives comes from the *sense update interval* the scheduler grants, not from this rate. The rates exist so that the behaviour can be tuned without touching the scheduler, and both being equal and very large means the system currently behaves as a latch with hysteresis.

## Exported units

- **`Vision`** — owns the candidate list and the per-candidate items. Constructed against an owning entity, which is excluded from its own traces. A release-case participant.
- **`query`** — gather the potentially visible set from a frustum. See the implementation twin.
- **`update`** — reconcile the potentially visible set against last frame's and run the traces.
- **`clear`** — forget everything.
- **`is_relevant`** — the game decides which entity classes this sense cares about at all. Mandatory.
- **`material_transparency`** — the game reports how much light a surface passes, given the entity and the surface element hit. Mandatory, and it is what makes glass, foliage and smoke partially see-through to the AI.
- **`get_visible`** — the currently-seen set: every candidate whose confidence is positive.
- **`get_vis_point`** — where on a seen target it was last seen. Asserts the target is currently seen, so the caller must have checked.
- **`on_object_released`** — drop a destroyed entity from all four lists.

**Notes** — The confidence is clamped to −0.5 at the bottom but 1 at the top (see the implementation), so losing sight of something decays only halfway to certainty-of-absence. An entity that was seen and is now hidden stays "recently seen" rather than becoming "never seen", which is the state the AI's search behaviour wants.

**Notes** — The position-granularity constant is declared and the early-out that would use it is commented out in the implementation. It records an intent — skip the trace when neither the viewer nor the target has moved appreciably — that is not currently realized.
