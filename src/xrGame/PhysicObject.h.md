# src/xrGame/PhysicObject.h

> Declares the generic physics prop and the buffer shape its multiplayer interpolation reads from, implemented in [`PhysicObject.cpp`](PhysicObject.cpp.md).

**Needs** — [`GameObject.h`](GameObject.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`PhysicsSkeletonObject.h`](PhysicsSkeletonObject.h.md) · [`PHSkeleton.h`](PHSkeleton.h.md) · [`animation_script_callback.h`](animation_script_callback.h.md) · [`xrServerEntities/xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md)
**Used by** — [`DestroyablePhysicsObject.h`](DestroyablePhysicsObject.h.md) · [`PhysicObject.cpp`](PhysicObject.cpp.md) · [`PhysicObject_script.cpp`](PhysicObject_script.cpp.md) · [`doors_door.cpp`](doors_door.cpp.md) · [`script_game_object4.cpp`](script_game_object4.cpp.md) · [`script_game_object_script3.cpp`](script_game_object_script3.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CPhysicObject`, the movable prop that joins the physics-shell holder to the
breakable-skeleton mixin. Substance is in [`PhysicObject.cpp`](PhysicObject.cpp.md).

It also declares the two records the network path needs, which are worth naming here
because they are the only part of the declaration that is not a method list:

- `net_update_PItem` — one timestamped authoritative pose sample (timestamp plus the
  physics net state: position, orientation, velocities, forces, enabled flag).
- `net_updatePhData` — the receive buffer: a double-ended queue of those samples plus the
  start and end clock values of the interpolation window currently being played out.

Exported units:

- `CPhysicObject` — the prop itself.
- The lifecycle overrides: `Load`, `net_Spawn`, `CreatePhysicsShell`, `net_Destroy`,
  `shedule_Update`, `UpdateCL`, `net_Save`, `net_SaveRelevant`, `net_Export`, `net_Import`.
- `Interpolate` / `interpolate_states` / `CalculateInterpolationParams` — the multiplayer
  pose blend.
- `PH_B_CrPr` / `PH_I_CrPr` / `PH_A_CrPr` — the three hooks the physics step calls around
  its correction and prediction phases.
- The script-facing door and animation surface: `run_anim_forward`, `run_anim_back`,
  `stop_anim`, `anim_time_get`, `anim_time_set`, `play_bones_sound`, `stop_bones_sound`,
  `set_door_ignore_dynamics`, `unset_door_ignore_dynamics`, `get_door_vectors`.
- `get_collision_hit_callback` / `set_collision_hit_callback`, `is_ai_obstacle`,
  `UsedAI_Locations`.

## Notes

The declaration carries an inventory-item flag enumeration (droppable, takeable,
tradeable, belt/rucksack placement, quest item, and two interpolation bits) even though a
prop is not an inventory item; only the two interpolation bits are used here. A rebuild
should give the prop the two state bits it needs and leave the inventory flags with
inventory items.
