# src/xrGame/ai/monsters/control_path_builder_base.h

> The default driver of the path channel: it turns "go there" or "get away from there" into a target the navigation stack can actually reach, and decides when a path is stale, finished or hopeless.

**Needs** — [`control_combase.h`](control_combase.h.md) · [`ai_monster_defs.h`](ai_monster_defs.h.md) · [`movement_manager_space.h`](../../movement_manager_space.h.md) · [`control_path_builder.h`](control_path_builder.h.md) · [`cover_evaluators.h`](../../cover_evaluators.h.md)
**Used by** — [`base_monster.cpp`](basemonster/base_monster.cpp.md) · [`base_monster_script.cpp`](basemonster/base_monster_script.cpp.md) · [`bloodsucker_vampire_execute_inline.h`](bloodsucker/bloodsucker_vampire_execute_inline.h.md) · [`burer_state_attack_run_around_inline.h`](burer/burer_state_attack_run_around_inline.h.md) · [`chimera.cpp`](chimera/chimera.cpp.md) · [`chimera_attack_state_inline.h`](chimera/chimera_attack_state_inline.h.md) · [`control_animation_base.cpp`](control_animation_base.cpp.md) · [`control_animation_base_accel.cpp`](control_animation_base_accel.cpp.md) · [`control_animation_base_update.cpp`](control_animation_base_update.cpp.md) · [`control_jump.cpp`](control_jump.cpp.md) · [`control_path_builder_base.cpp`](control_path_builder_base.cpp.md) · [`control_path_builder_base_inline.h`](control_path_builder_base_inline.h.md) · [`control_path_builder_base_path.cpp`](control_path_builder_base_path.cpp.md) · [`control_path_builder_base_set.cpp`](control_path_builder_base_set.cpp.md) · _and 3 more_
**Tier floor** — T2: runs every frame and drives bounded searches and cover queries

## Purpose

The base element of the path channel, and the piece of the chapter that does the most
thinking. Its implementation is split four ways by topic and the state is declared here,
so this twin carries the state record and the shape of the problem:
[`control_path_builder_base.cpp`](control_path_builder_base.cpp.md) has the lifecycle and
the event handling, [`control_path_builder_base_set.cpp`](control_path_builder_base_set.cpp.md)
the target-setting surface,
[`control_path_builder_base_update.cpp`](control_path_builder_base_update.cpp.md) the
per-frame state machine, and
[`control_path_builder_base_path.cpp`](control_path_builder_base_path.cpp.md) the target
resolution — the interesting part. Inline setters are in
[`control_path_builder_base_inline.h`](control_path_builder_base_inline.h.md).

The problem: the creature's state layer says "go to this position". That position may be
inside a wall, outside the creature's restrictors, on the wrong side of a closed door, or
simply the creature's own feet. This element's job is to turn the *set* target into a
*found* target that the navigation stack can path to, to notice when the found target has
gone stale, and to notice when the creature has been failing to path for long enough that
it should try something else entirely.

## State

```text
RECORD PathBuilderBase
  # --- what the state layer asked for, published into the channel payload ---
  enable, try_min_time, use_dest_orient, extrapolate : bool
  dest_dir            : vector
  path_type           : one of { level, game, patrol }
  velocity_mask       : int (bit set)
  desirable_mask      : int (bit set)
  game_graph_target   : int
  reset_actuality     : bool

  # --- the two targets ---
  target_set          : { position : vector, node : int }   # what was asked for
  target_found        : { position : vector, node : int }   # what we will actually path to
  target_actual       : bool          # the found target still answers the set target
  target_type         : one of { move_to, retreat_from }

  # --- timing and failure ---
  rebuild_time              : int (ms)  # minimum gap between replans of a valid path
  distance_to_path_end      : real      # how close counts as arrived
  last_time_target_set      : int (ms)
  last_time_dir_set         : int (ms)
  time_path_updated_external: int (ms)
  time_global_failed_started: int (ms)
  failed                    : bool      # a failure was observed since the last state pass
  path_end                  : bool      # the last waypoint has been passed

  # --- cover approach, when enabled ---
  cover.use_covers  : bool
  cover.min_dist, cover.max_dist, cover.deviation, cover.radius : real
  cover_approach    : CoverEvaluatorCloseToEnemy

  state : int (bit set)   # path_valid | wait_new_path | path_end | no_path | path_failed
```

**Invariants** — the two targets are not interchangeable and the distinction is the
element's core idea. `target_set` is an intention and may be unreachable; `target_found`
is a promise that a path can be planned. `target_actual` says the promise still answers
the intention, and it is cleared the moment the intention changes.

`path_valid` is exclusive with `path_failed`: observing a failure clears the valid and
waiting bits together. The other bits combine.

The *global failure* window is three seconds. While the creature is inside it, target
resolution abandons the intention entirely and picks random reachable points instead —
see [`control_path_builder_base_path.cpp`](control_path_builder_base_path.cpp.md). That
window is the difference between a creature that sticks on a doorframe and one that
visibly gives up and wanders.

## Exported units

**Lifecycle and events** (see [`control_path_builder_base.cpp`](control_path_builder_base.cpp.md)):
`reinit`, `reset`, `on_event`, `on_start_control`, `on_stop_control`, `on_path_built`,
`on_path_updated`, `on_path_end`, `travel_point_changed`, `global_failed`,
`set_dest_direction`, `set_target_accessible`, `detour_graph_points`.

**Setting a target** (see [`control_path_builder_base_set.cpp`](control_path_builder_base_set.cpp.md)):
`prepare_builder`, `set_target_point` (by position-and-node or by node alone),
`set_retreat_from_point`.

**The per-frame pass** (see [`control_path_builder_base_update.cpp`](control_path_builder_base_update.cpp.md)):
`update_frame`, `update_path_builder_state`, `update_target_point`,
`set_path_builder_params`.

**Resolving a target** (see [`control_path_builder_base_path.cpp`](control_path_builder_base_path.cpp.md)):
`target_point_need_update`, `find_target_point_set`, `find_target_point_failed`,
`find_node`.

**Parameter setters** — the one-line ones are in
[`control_path_builder_base_inline.h`](control_path_builder_base_inline.h.md);
`enable_path`, `disable_path`, `extrapolate_path`, `set_try_min_time`,
`set_use_dest_orient`, `set_level_path_type`, `set_game_path_type`,
`set_patrol_path_type`, `set_velocity_mask`, `set_desirable_mask` are trivial field
writes declared here.

**Queries** — `enabled`, `is_target_actual`, `get_target_found`, `get_target_found_node`,
`get_target_set`.

`pre_update` is declared and never defined anywhere; it is dead and must not be
reproduced.
