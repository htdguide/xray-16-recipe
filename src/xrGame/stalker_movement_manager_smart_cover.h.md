# src/xrGame/stalker_movement_manager_smart_cover.h

> Declares the top layer of a stalker's movement manager — the layer that knows
> how to enter, traverse and leave a **smart cover**.

**Needs** — [`stalker_movement_manager_obstacles.h`](stalker_movement_manager_obstacles.h.md) · [`stalker_movement_params.h`](stalker_movement_params.h.md) · [`smart_cover.h`](smart_cover.h.md) · [`smart_cover_loophole.h`](smart_cover_loophole.h.md) · [`smart_cover_transition.hpp`](smart_cover_transition.hpp.md) · [`smart_cover_animation_selector.h`](smart_cover_animation_selector.h.md)
**Used by** — [`00dummy_tester.cpp`](../dummy/00dummy_tester.cpp.md) · [`ai_stalker.cpp`](ai/stalker/ai_stalker.cpp.md) · [`ai_stalker_feel.cpp`](ai/stalker/ai_stalker_feel.cpp.md) · [`ai_stalker_script_entity.cpp`](ai/stalker/ai_stalker_script_entity.cpp.md) · [`cover_evaluators.cpp`](cover_evaluators.cpp.md) · [`death_anims_predicates.cpp`](death_anims_predicates.cpp.md) · [`doors_actor.cpp`](doors_actor.cpp.md) · [`dynamic_obstacles_avoider.cpp`](dynamic_obstacles_avoider.cpp.md) · [`script_game_object_smart_covers.cpp`](script_game_object_smart_covers.cpp.md) · [`sight_action.cpp`](sight_action.cpp.md) · [`sight_manager.cpp`](sight_manager.cpp.md) · [`sight_manager_target.cpp`](sight_manager_target.cpp.md) · [`smart_cover_animation_planner.cpp`](smart_cover_animation_planner.cpp.md) · [`smart_cover_animation_selector.cpp`](smart_cover_animation_selector.cpp.md) · _and 35 more_
**Tier floor** — T2: pure game logic over graph search and animation selection; nothing here touches a device or a byte layout.

## Purpose

A stalker's movement manager is a stack of layers, each adding one concern to the one
below: a base layer that turns a destination into a path, an obstacles layer that keeps
the path clear of moving objects, and this layer, which adds the idea that the creature
may be *inside a piece of authored furniture* — a smart cover — where position is no
longer a point on the navigation mesh but a named **loophole**, and where getting from
one loophole to another means playing a transition animation rather than walking.

The class is declared here and implemented across three files: the lifecycle and the
per-frame update in [`stalker_movement_manager_smart_cover.cpp`](stalker_movement_manager_smart_cover.cpp.md),
the field-of-view and range questions in
[`stalker_movement_manager_smart_cover_fov_range.cpp`](stalker_movement_manager_smart_cover_fov_range.cpp.md),
and everything about loophole paths in
[`stalker_movement_manager_smart_cover_loopholes.cpp`](stalker_movement_manager_smart_cover_loopholes.cpp.md).
The split is by topic, not by necessity; a rebuild may keep them together.

## State

The layer owns the *route through a cover* and the transition currently being played.
Everything else about where the creature is going lives in the two
[`stalker_movement_params`](stalker_movement_params.h.md) records — `current` and `target` —
which the base layer already holds.

```text
RECORD SmartCoverMovementState
  path                  : list<text>     # loophole identifiers, front = where we are,
                                         # back = where we are going. The pseudo-loopholes
                                         # "enter" and "exit" bracket a route that crosses
                                         # the cover's boundary.
  temp_path             : list<text>     # scratch for one candidate route; reused to keep
                                         # route search allocation-free per frame
  current_transition    : optional<TransitionAction>   # the edge being traversed now
  current_transition_animation : optional<TransitionAnimation>
                                         # invariant: set exactly when current_transition
                                         # is set, and is that transition's animation
  target_selector       : TargetSelector  # decides what the creature aims at while inside
  animation_selector    : AnimationSelector # drives the in-cover animation planner
  property_storage      : PropertyStorage # the planner world-state this layer writes into
  apply_loophole_direction_distance : real = 4.0
                                         # within this distance of the next loophole, stop
                                         # looking along the path and start looking along
                                         # the loophole's own enter direction
  enter_animation       : MotionRef
  enter_cover_id        : text = ""      # the cover being entered while the enter
  enter_loophole_id     : text = ""      # animation plays, before `current` names it
  entering_with_animation : bool
  non_animated_loophole_change : bool    # script has asked for walked, not animated,
                                         # loophole changes
  default_behaviour     : bool
  check_can_kill_enemy  : bool
  combat_behaviour      : bool
  ray_query_scratch     : RayQueryResults # reused between visibility test picks
```

Invariants worth stating because nothing else states them:

- `path` is never empty once a cover is involved; a single-element path means "we are
  already at the only loophole we want".
- `path.front()` always names where the creature *is* — the current loophole, or the
  pseudo-loophole `enter` when the creature is still outside. Any other value means the
  path is stale and must be rebuilt.
- `enter_cover_id` / `enter_loophole_id` exist only because during the enter animation the
  creature is neither outside nor properly inside: the field-of-view questions must still
  be answerable, and the `current` params do not yet name the cover.

## Exported surface

Lifecycle, overriding the layer below — `reinit`, `update(time_delta)`,
`on_frame(movement_control, out_position)`, `remove_links(object)`, and
`cleanup_after_animation_selector`.

Where am I — `in_smart_cover`, `entering_smart_cover_with_animation`, `default_behaviour`,
`combat_behaviour` (get and set), `check_can_kill_enemy` (get and set).

Can I shoot from here — `enemy_in_fov`, `in_fov(cover, loophole, position)`,
`in_range(cover, loophole, position)`, `in_current_loophole_fov(position)`,
`in_current_loophole_range(position)`, `apply_loophole_direction_distance` (get and set).
Contracts in [`…_fov_range.cpp`](stalker_movement_manager_smart_cover_fov_range.cpp.md).

Moving through the cover — `current_transition`, `exit_transition`, `go_next_loophole`,
`start_non_animated_loophole_change`, `stop_non_animated_loophole_change`,
`position_to_cover_from`. Contracts in
[`…_loopholes.cpp`](stalker_movement_manager_smart_cover_loopholes.cpp.md).

What the creature does while inside — `target_selector` (get, and set from a script
callback), and the five fixed targets `target_idle`, `target_lookout`, `target_fire`,
`target_fire_no_lookout`, `target_default(value)`.

Idle and lookout timing — `idle_min_time`, `idle_max_time`, `lookout_min_time`,
`lookout_max_time`, each a get/set pair forwarded to the in-cover animation planner.

The animation side — `animation_selector`, `property_storage(storage)`.

**Notes**

The class is a deep inheritance chain in the original (base → obstacles → smart cover),
and the chain is load-bearing only in that each layer must run *before* the one below on
the way down and *after* it on the way up: the smart-cover layer decides a destination,
the obstacles layer clears it, the base layer walks it. A rebuild can express that as
composition. What must not change is the order.
