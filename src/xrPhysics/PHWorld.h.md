# src/xrPhysics/PHWorld.h

> Declares the physics world: the fixed-timestep accumulator, the four object registries, and the global handle every other module reaches physics through.

**Needs** — [`PHWorld.cpp`](PHWorld.cpp.md) · [`Physics.h`](Physics.h.md) · [`PHUpdateObject.h`](PHUpdateObject.h.md) · [`IPHWorld.h`](IPHWorld.h.md) · [`PHObject.h`](PHObject.h.md) · [`xrEngine/pure.h`](../xrEngine/pure.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`ActorCameraCollision.cpp`](ActorCameraCollision.cpp.md) · [`ElevatorState.cpp`](ElevatorState.cpp.md) · [`GeometryBits.cpp`](GeometryBits.cpp.md) · [`MovementBoxDynamicActivate.cpp`](MovementBoxDynamicActivate.cpp.md) · [`PHActivationShape.cpp`](PHActivationShape.cpp.md) · [`PHCaptureInit.cpp`](PHCaptureInit.cpp.md) · [`PHDisabling.cpp`](PHDisabling.cpp.md) · [`PHFracture.cpp`](PHFracture.cpp.md) · [`PHGeometryOwner.cpp`](PHGeometryOwner.cpp.md) · [`PHInterpolation.cpp`](PHInterpolation.cpp.md) · [`PHObject.cpp`](PHObject.cpp.md) · [`PHShellActivate.cpp`](PHShellActivate.cpp.md) · [`PHSimpleCalls.cpp`](PHSimpleCalls.cpp.md) · [`PHSimpleCharacter.cpp`](PHSimpleCharacter.cpp.md) · _and 8 more_
**Tier floor** — T2: registries and a time accumulator.

## Purpose

Declares the surface implemented in [`PHWorld.cpp`](PHWorld.cpp.md) and
[`PHWorldScript.cpp`](PHWorldScript.cpp.md). The world implements the engine's per-frame interface
(it registers itself as a frame consumer at creation) and the module-facing interface in
[`IPHWorld.h`](IPHWorld.h.md), which is how `xrGame` and `xrEngine` reach physics without depending
on this module's types — the cycle-break described in
[§7 of the system requirements](../../SYSTEM-REQUIREMENTS.md#7-build-order).

There is exactly one world, reached through a module-global handle. That is the service-locator
pattern the preface calls out; a rebuild should inject it.

## Exported units

- `CPHWorld` — the world. Every algorithm is in [`PHWorld.cpp`](PHWorld.cpp.md).
- `CPHMesh` — a one-field wrapper holding the shape handle that represents the whole level's
  static triangle soup to the collider. Created once per level, destroyed with the world. Its
  entire job is to give the level mesh the same shape identity as any other collidable, so the
  contact generator needs no special case for "versus the world".
- `V_PH_WORLD_STATE` — a snapshot of every simulated element's network state, produced by
  `get_state` for the multiplayer comparison path.
- `ph_world` — the module-global handle; `inl_ph_world()` is its dereference.

## Notes

The header carries one constant worth naming: the sound cache size, and one unused delay-smoothing
group (`m_delay`, `m_previous_delay`, `m_reduce_delay`, `m_update_delay_count`) that is initialized
and never read. The evident intent was to spread a frame's step burst across several frames when
the accumulator falls behind; it was not finished. A rebuild should not reproduce the fields, but
should notice the problem they were aimed at — see the burst discussion in
[`PHWorld.cpp`](PHWorld.cpp.md).
