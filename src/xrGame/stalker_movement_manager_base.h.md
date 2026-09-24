# src/xrGame/stalker_movement_manager_base.h

> Declares the layer that turns "walk there, crouched, alarmed" into a path, a speed and a body orientation.

**Needs** — [`stalker_movement_manager_base.cpp`](stalker_movement_manager_base.cpp.md) · [`stalker_movement_manager_base_inline.h`](stalker_movement_manager_base_inline.h.md) · [`movement_manager.h`](movement_manager.h.md) · [`stalker_movement_params.h`](stalker_movement_params.h.md) · [`stalker_movement_manager_space.h`](stalker_movement_manager_space.h.md) · [`ai_monster_space.h`](ai_monster_space.h.md)
**Used by** — [`stalker_movement_manager_base.cpp`](stalker_movement_manager_base.cpp.md) · [`stalker_movement_manager_base_inline.h`](stalker_movement_manager_base_inline.h.md) · [`stalker_movement_manager_obstacles.h`](stalker_movement_manager_obstacles.h.md)
**Tier floor** — T2: one instance per creature, updated every scheduler tick.

## Purpose

Declares the surface implemented in
[`stalker_movement_manager_base.cpp`](stalker_movement_manager_base.cpp.md). It is the
bottom of a three-layer stack: this base does paths and speeds, the obstacle layer above it
handles other creatures in the way, and the smart-cover layer above that handles authored
cover. The AI layer above all three only ever talks in terms of
[movement params](stalker_movement_params.h.md).

## State

```text
velocities        : ref VelocityCollection   # the speed table named by the creature's section
danger_head_speed : real                     # how fast the head turns when alarmed
last_turn_index   : optional<int>            # the path point a turn-in-place was last done at
current, target   : MovementParams           # the whole point: what is, and what is wanted
head              : BoneRotation             # current and target head angles, plus a speed
force_update      : bool                     # update even while standing still
the object-on-the-way memo : the last query's object, both positions, distance and answer
```

## Exported units

Lifecycle — `Load`, `reload`, `reinit`, `initialize`, `update(time_delta)`.

Setters the AI layer uses: `set_desired_position`, `set_desired_direction`,
`set_body_state`, `set_movement_type`, `set_mental_state`, `set_path_type`,
`set_detail_path_type`, `set_level_dest_vertex`, `set_head_orientation`,
`set_nearest_accessible_position` in two forms, `force_update`, `danger_head_speed`,
`setup_speed_from_animation`.

Readers: `head_orientation`, `desired_position`, `desired_direction`, `body_state`,
`target_body_state`, `movement_type`, `target_movement_type`, `mental_state`,
`target_mental_state`, `path_type`, `detail_path_type`, `current_params`, `target_params`,
`object`, `path_direction_angle`, `turn_in_place`, `speed(direction)`.

Notifications it overrides: `on_travel_point_change`, `on_restrictions_change`,
`on_build_path`, `remove_links`.

Queries: `is_object_on_the_way(object, distance)` and its refresh helper.

**Notes** — the header carries two assertions worth naming because they are invariants of
the whole layer, not debugging aids: a creature may never be simultaneously *at ease* and
*crouched*. Both the posture setter and the mental-state setter enforce it, and the pairing
is real — the relaxed animation set has no crouched variants, so the combination has no
pose to play.
