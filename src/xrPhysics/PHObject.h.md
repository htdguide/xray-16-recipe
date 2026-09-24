# src/xrPhysics/PHObject.h

> Declares the base every simulated physics entity derives from: the unit the world steps, collides and freezes.

**Needs** — [`PHItemList.h`](PHItemList.h.md) · [`PHIsland.h`](PHIsland.h.md) · [`PHObject.cpp`](PHObject.cpp.md) · [`xrCDB/ISpatial.h`](../xrCDB/ISpatial.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`PHActivationShape.cpp`](PHActivationShape.cpp.md) · [`PHActivationShape.h`](PHActivationShape.h.md) · [`PHCharacter.h`](PHCharacter.h.md) · [`PHCollideValidator.cpp`](PHCollideValidator.cpp.md) · [`PHCollideValidator.h`](PHCollideValidator.h.md) · [`PHObject.cpp`](PHObject.cpp.md) · [`PHShell.cpp`](PHShell.cpp.md) · [`PHShell.h`](PHShell.h.md) · [`PHShellActivate.cpp`](PHShellActivate.cpp.md) · [`PHSimpleCharacter.cpp`](PHSimpleCharacter.cpp.md) · [`PHSplitedShell.cpp`](PHSplitedShell.cpp.md) · [`PHStaticGeomShell.cpp`](PHStaticGeomShell.cpp.md) · [`PHStaticGeomShell.h`](PHStaticGeomShell.h.md) · _and 5 more_
**Tier floor** — T2: an abstract participant in the step loop; nothing here touches a byte layout.

## Purpose

Declares the surface implemented in [`PHObject.cpp`](PHObject.cpp.md). A *physics object* is one
independently-schedulable unit of simulation: a rigid-body shell, a character capsule, or a static
collision proxy. The world holds a list of them and drives each through the same four-phase step.

This header is substantive in one respect only: it fixes **what a subclass must supply** for the
world's step loop to work at all. Those demands are the contract; the rest is delegation.

## What a subclass must supply

```text
# Identity and geometry
FUNCTION collision_shape() -> shape_handle      # the composite shape offered to the collider
FUNCTION recompute_bounds()                     # refresh the spatial sphere and half-extents
FUNCTION cast_type() -> {undefined, shell, character, static_shell}

# The four step phases, called in this order by the world
FUNCTION tune(step)                             # pre-solve: apply accumulated contact effects
FUNCTION data_update(step)                      # post-solve: clamp, damp, disable, interpolate

# Contact shaping
FUNCTION init_contact(contact, INOUT do_collide, material_a, material_b)

# Network state
FUNCTION element_count() -> int
FUNCTION element_sync(index) -> synchronizable
```

Optional overrides with useful defaults: `collide` (the whole broadphase-plus-contact pass),
`near_callback` (notification that another object is close), `cut_velocity` (an external velocity
clamp), `move_storage` (the swept-motion ray set), and the two visibility hooks
`on_processing_activate` / `on_processing_deactivate`.

## Exported units

- `CPHObject` — the base itself; see [`PHObject.cpp`](PHObject.cpp.md) for every algorithm.
- `ECastType` — the tag a caller uses instead of a dynamic cast when it must know whether a
  contact partner is a character.
- `CollideCallback` — the shape of the pairwise collision entry point, so a collision partner can
  be handed to the contact generator without naming the generator's module.
- `PH_OBJECT_STORAGE` / `PH_OBJECT_I` — the intrusive-list alias pair for this type
  (see [`PHItemList.h`](PHItemList.h.md)).
