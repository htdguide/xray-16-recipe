# src/xrGame/sight_manager.h

> Declares the per-creature aiming manager: the currently chosen look order, the smoothed bone rotations it produces, and the torso-twist limits that make a creature turn its whole body.

**Needs** — [`setup_manager.h`](setup_manager.h.md) · [`sight_control_action.h`](sight_control_action.h.md) · [`sight_manager_space.h`](sight_manager_space.h.md) · [`xrServer_Space.h`](../xrServerEntities/xrServer_Space.h.md)
**Used by** — [`base_monster_script.cpp`](ai/monsters/basemonster/base_monster_script.cpp.md) · [`ai_stalker.cpp`](ai/stalker/ai_stalker.cpp.md) · [`ai_stalker_feel.cpp`](ai/stalker/ai_stalker_feel.cpp.md) · [`ai_stalker_script_entity.cpp`](ai/stalker/ai_stalker_script_entity.cpp.md) · [`script_game_object3.cpp`](script_game_object3.cpp.md) · [`script_game_object4.cpp`](script_game_object4.cpp.md) · [`sight_action.cpp`](sight_action.cpp.md) · [`sight_manager.cpp`](sight_manager.cpp.md) · [`sight_manager_inline.h`](sight_manager_inline.h.md) · [`sight_manager_target.cpp`](sight_manager_target.cpp.md) · [`smart_cover_loophole_planner_actions.cpp`](smart_cover_loophole_planner_actions.cpp.md) · [`smart_cover_planner_actions.cpp`](smart_cover_planner_actions.cpp.md) · [`stalker_alife_actions.cpp`](stalker_alife_actions.cpp.md) · [`stalker_alife_task_actions.cpp`](stalker_alife_task_actions.cpp.md) · _and 15 more_
**Tier floor** — T2: per-frame rotation state for every animated creature

## Purpose

Declares the surface implemented in [`sight_manager.cpp`](sight_manager.cpp.md),
[`sight_manager_target.cpp`](sight_manager_target.cpp.md) and
[`sight_manager_inline.h`](sight_manager_inline.h.md). It is an instance of the generic
action-selection manager over look orders, extended with everything aiming needs that
action selection does not know about.

## Exported units

- **The class** — a selector over wrapped look orders
  ([`sight_control_action.h`](sight_control_action.h.md)) for one creature.
- **Two enumerations** — *aiming type* (none, weapon-driven, head-driven) selects which
  bone solver produces the target rotations; *animation frame type* (none, start, end)
  says which end of the currently playing clip the solver should match.
- **Lifecycle** — `Load` (nothing), `reinit`, `reload` (reads the torso limits from the
  creature's configuration section).
- **`setup`** — issue a look order. Four forms: one taking a built order, and three
  forwarding one, two or three arguments to the order's constructors so callers never
  name the order type.
- **`update`** — the scheduled half: run the turning-in-place rule, then let the selector
  execute the chosen order.
- **`Exec_Look`** — the per-frame half: normalize, clamp, advance the angles toward their
  targets, compute the bone rotations, and write the creature's world orientation.
- **Angle solvers** — `SetPointLookAngles`, `SetFirePointLookAngles`, `SetDirectionLook`,
  the two `SetLessCoverLook` forms, `GetDirectionAngles`,
  `GetDirectionAnglesByPrevPositions`, `vfValidateAngleDependency`. Substance in
  [`sight_manager_target.cpp`](sight_manager_target.cpp.md).
- **`aiming_position` / `object_position`** — where the creature is aiming in world space,
  per sight type; read by the weapon and the bone solvers.
- **`current_spine_rotation` / `current_shoulder_rotation` / `current_head_rotation`** —
  the additive rotations the animation layer applies on top of the playing clip.
- **`bone_aiming`** — arm or disarm the bone solver, naming the clip and which end of it.
- **`enable` / `enabled`** — suspend aiming entirely, e.g. while a scripted animation owns
  the creature.
- **`turning_in_place`** — whether the creature is currently rotating its feet to catch up
  with its head.
- **`use_torso_look`** — whether the chosen order wants the torso to follow.
- **`remove_links`** — forwarded to every held order when an entity dies.

## State

```text
RECORD sight_manager
  current  : { spine, shoulder, head } each { rotation : matrix, factor : real }
  target   : { spine, shoulder, head } each { rotation : matrix }
  animation_id      : text            # clip the bone solver matches against
  aiming_type       : none | weapon | head
  animation_frame   : none | start | end
  max_left_angle    : real            # torso twist limits, radians, from configuration
  max_right_angle   : real
  enabled           : bool
  turning_in_place  : bool
```

**Invariants** — `current` carries a blend factor per bone and `target` does not: the
factors are only used by the no-aimer path, where the rotation is *derived* from the
head/body angle difference rather than solved for. When an aimer is armed the factors are
stale and unread. The two representations share the same fields by inheritance in the
original; a rebuild should make them two distinct records, because conflating them is what
makes the factors look live when they are not.

## Notes

The left and right torso limits are *asymmetric* by default — a creature may twist further
to one side than the other. That is anatomical, it comes from configuration per creature
section, and it is what decides when the creature gives up twisting and turns its feet.
