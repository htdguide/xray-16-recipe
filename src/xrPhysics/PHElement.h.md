# src/xrPhysics/PHElement.h

> Declares one simulated rigid body: its shapes, its mass, its forces, its bone binding, its network state and its breakability.

**Needs** — [`PHElement.cpp`](PHElement.cpp.md) · [`PHElementInline.h`](PHElementInline.h.md) · [`PHGeometryOwner.h`](PHGeometryOwner.h.md) · [`PHInterpolation.h`](PHInterpolation.h.md) · [`PHFracture.h`](PHFracture.h.md) · [`PHDisabling.h`](PHDisabling.h.md) · [`Geometry.h`](Geometry.h.md) · [`PHDefs.h`](PHDefs.h.md) · [`PhysicsCommon.h`](PhysicsCommon.h.md) · [`xrServerEntities/PHSynchronize.h`](../xrServerEntities/PHSynchronize.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHCapture.cpp`](PHCapture.cpp.md) · [`PHCaptureInit.cpp`](PHCaptureInit.cpp.md) · [`PHElement.cpp`](PHElement.cpp.md) · [`PHElementInline.h`](PHElementInline.h.md) · [`PHElementNetState.cpp`](PHElementNetState.cpp.md) · [`PHFracture.cpp`](PHFracture.cpp.md) · [`PHFracture.h`](PHFracture.h.md) · [`PHJoint.cpp`](PHJoint.cpp.md) · [`PHJoint.h`](PHJoint.h.md) · [`PHShell.cpp`](PHShell.cpp.md) · [`PHShell.h`](PHShell.h.md) · [`PHShellActivate.cpp`](PHShellActivate.cpp.md) · [`PHShellBuildJoint.h`](PHShellBuildJoint.h.md) · [`PHShellNetState.cpp`](PHShellNetState.cpp.md) · _and 4 more_
**Tier floor** — T1: the declared surface hands mass tensors and body handles across the dynamics-library boundary.

## Purpose

Declares the surface implemented in [`PHElement.cpp`](PHElement.cpp.md),
[`PHElementNetState.cpp`](PHElementNetState.cpp.md) and [`PHElementInline.h`](PHElementInline.h.md).

An *element* is exactly one rigid body: one bone of a skeleton, or the whole of a simple object.
Elements never exist alone — they always belong to a *shell* ([`PHShell.h`](PHShell.h.md)), which
owns the island they share and the joints between them. The split exists because almost everything
the game does to physics is addressed to a whole object (apply an impulse, set a transform) while
almost everything the simulation does is per body.

The type is assembled from four roles, and a rebuild can keep them separate: the public
element interface the game layer holds; the network-synchronizable state; the sleep-detection
accumulator ([`PHDisabling.h`](PHDisabling.h.md)); and the shape composite
([`PHGeometryOwner.h`](PHGeometryOwner.h.md)).

## Exported units

The public surface, grouped by what it is for. Contracts are in the implementation twin.

**Shapes** — `add_sphere`, `add_box`, `add_cylinder`, `add_shape` (with and without an offset),
`add_geom`, `remove_geom`, `geometry(index)`, `last_geom`, `has_geoms`, `number_of_geoms`,
`set_material`, `get_extensions`, `get_max_area_dir`, `get_radius`, `get_point_vel`.

**Mass** — `set_mass`, `set_density`, `set_mass_mc` and `set_density_mc` (which also relocate the
mass centre), `set_box_mass`, `add_mass` (fold one authored bone shape's mass in), `set_inertia`,
`add_inertia`, `mass_center`, `local_mass_center`, `set_local_mass_center`, `get_mass`,
`get_density`, `get_mass_tensor`, `reset_mass`, `re_adjust_mass_positions`.

**Sleep** — `disable`, `re_enable`, `enable`, `is_enabled`, `freeze`, `unfreeze`,
`enabled_state_on_step`, `set_disable_params`.

**Dynamics** — `apply_force` (by vector or components), `apply_impulse`,
`apply_impulse_vs_mass_center`, `apply_impulse_vs_global_frame`, `apply_impulse_trace` (an impulse
attributed to a named bone), `apply_impact`, `apply_gravity_accel`, `set_force`, `set_torque`,
`get_force`, `get_torque`, `get_linear_vel`, `get_angular_vel`, `set_linear_vel`,
`set_angular_vel`, `set_air_resistance`, `set_dynamic_limits`, `set_dynamic_scales`,
`cut_velocity`, `set_apply_by_gravity`, `fix`, `release_fixed`, `is_fixed`, `set_animated`.

**Placement** — `set_transform`, `transform_position`, `set_global_position_dynamic`,
`get_global_position_dynamic`, `get_global_transform_dynamic`, `interpolate_global_transform`,
`interpolate_global_position`, `get_quaternion`, `set_quaternion`, and the two frame conversions
`cv2obj_xform` / `cv2bone_xform`.

**Animation binding** — `set_bone_callback`, `clear_bone_callback`,
`set_bone_callback_overwrite`, `bones_callback`, `static_root_bones_callback`, `to_bone_pos`,
`anim_to_vel`, `get_anim_bone_pos`, `bone_gl_pos`.

**Step participation** — `ph_tune`, `ph_data_update`, `update`.

**Network** — `get_state`, `set_state`, `net_import`, `net_export`.

**Breakability** — `is_breakable`, `set_geom_fracturable`, `fracture(index)`, `split_process`,
`pass_end_geoms`, `fractures_holder`, `delete_fractures_holder`, `clear_destroy_info`.

**Lifecycle** — the four `activate` overloads (from a transform, from a transform pair implying a
velocity, from the element's own recorded frame, or relative to a parent frame), `deactivate`,
`build`, `destroy`, `start`, `run_simulation`, `create_simul_base`, `re_init_dynamics`,
`preset_active`.

## Notes

The declared field set carries an annotation scheme in the original — each field is tagged with
whether it is *element* state, *blueprint* state, *step* state or *auxiliary*. The distinction the
tags are groping toward is real and useful to a rebuild: the description of a body (shapes, mass,
limits) is authored data that could be shared between instances, while the simulation state is
per-instance and per-step. The original does not act on the distinction; a rebuild could, and would
save a large amount of per-instance memory on a level with many copies of the same prop.

Several commented-out fields record a "safe state" (position, orientation, velocity) that was
maintained alongside the live one. That machinery survives in
[`PHValideValues.h`](PHValideValues.h.md) and is used by the character; for elements it was
abandoned in favour of the assertion-and-clamp approach in `ph_data_update`.
