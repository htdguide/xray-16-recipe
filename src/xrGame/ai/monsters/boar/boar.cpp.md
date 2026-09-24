# src/xrGame/ai/monsters/boar/boar.cpp

> The boar's definition: the table binding its animation clips to logical motions, velocities and actions, plus the two things that are its own — a head that tracks the enemy while the body runs, and a jump-turn.

**Needs** — [`boar.h`](boar.h.md) · [`boar_state_manager.h`](boar_state_manager.h.md) · [`base_monster.h`](../basemonster/base_monster.h.md) · [`control_animation_base.h`](../control_animation_base.h.md) · [`control_movement_base.h`](../control_movement_base.h.md) · [`monster_velocity_space.h`](../monster_velocity_space.h.md) · [`ai_monster_defs.h`](../ai_monster_defs.h.md)
**Used by** — [`boar.h`](boar.h.md)
**Tier floor** — T2: the head-tracking hook runs inside the animation layer's per-bone pass, which is a hot per-frame path over the skeleton

## Purpose

Almost all of this file is a *declaration of data*: which clip name prefix stands for each logical motion, which velocity profile that motion moves at, which posture it belongs to, how the creature gets from one posture to another, and which logical motion each abstract action plays. A rebuilder should read it as a table with a little code around it, and should expect to author the same table for every creature in the chapter.

The boar's own contributions are small and both are visible in play: its head turns toward what it is fighting independently of its body, and it can spin in place with a jump instead of a turn animation.

## State

```text
RECORD Boar EXTENDS BaseMonster, Controllable
  look_at_enemy     : bool      # gate for the head-tracking hook
  current_head_delta: real      # radians, the head's present extra yaw
  target_head_delta : real      # radians, where the head is being driven to
  head_turn_speed   : real      # radians per second; set to half a turn per second
```

**Invariants** — `current_head_delta` is only ever moved toward `target_head_delta` by the frame update, never assigned directly; the animation hook reads it and must never write it. Both are zero and `look_at_enemy` false until the creature is placed in the world.

## `Load`

**Contract** — Reads the creature's configuration section and builds its animation table. Called once per creature at spawn. Allocates the table; fails loudly if a clip the table names is absent from the model, since a missing clip means the creature would have no animation to play for an action it can reach.

```text
FUNCTION Load(section)
  base.Load(section)

  # Abilities. Friendly boars (a flag in the section) lose the charging attack.
  IF section does not define "is_friendly"
    enable_ability(run_attack)
  enable_ability(rotation_jump)

  # Conditional substitutions: while a flag is set, one motion stands in for another.
  substitute(when damaged,        run       -> run_damaged)
  substitute(when damaged,        walk      -> walk_damaged)
  substitute(when turning left while running,  run -> run_turn_left)
  substitute(when turning right while running, run -> run_turn_right)

  # Acceleration chains: which slow motion may ramp up into which fast one.
  load_acceleration_parameters(section)
  accel_chain(walk -> run)
  accel_chain(walk -> run_turn_left)
  accel_chain(walk -> run_turn_right)
  accel_chain(walk_damaged -> run_damaged)

  # The table proper: (logical motion, clip name prefix, variant, velocity profile,
  # posture, footstep effect set). Variant -1 means "count the variants in the model
  # and pick one at random each time"; a number pins one clip.
  FOR EACH row IN animation_table
    add_animation(row)

  # Posture transitions: getting from standing to lying plays a clip.
  transition(lie_down -> sleep     VIA lie_to_sleep,  chained = false)
  transition(standing -> sleep     VIA stand_lie_down, chained = true)
  transition(standing -> lying     VIA stand_lie_down, chained = false)
  transition(lying    -> standing  VIA lie_stand_up,   chained = false,
             skip when aggressive)   # a startled boar gets up without the animation

  # Action bindings: each abstract action the behaviour states ask for
  # resolves to one logical motion.
  FOR EACH (action, motion) IN action_table
    bind(action, motion)

  post_load(section)
```

**Notes** — Three table entries are worth naming because they are substitutions a rebuilder would not guess. Walking *backwards* is bound to the corpse-drag clip, and dragging is bound to the same one, because a boar only ever moves backwards while dragging something. Dying is bound to the idle clip — the boar has no death animation and falls to the physics simulation instead. "Look around" is the idle clip pinned to variant 2, i.e. one particular idle out of the set is reserved to mean scanning.

The footstep effect set (four named effects, one per direction of impact) is attached to every entry except plain standing idle. That is an authored inconsistency, not a rule.

## `reinit`

**Contract** — Re-arms the per-life state. Registers the jump-turn's data: the left and right spin clips (variant 0 of each), the angle beyond which a spin is used instead of a turn, and the flags saying the spin stops the creature dead and happens only once per decision.

```text
FUNCTION reinit()
  base.reinit()
  configure_rotation_jump(
      left_clip  = "stand_jump_left_0",
      right_clip = "stand_jump_right_0",
      threshold  = five sixths of a half-turn,   # ~150 degrees
      flags      = { stop_at_once, rotate_once })
```

**Notes** — The threshold is the interesting number: below roughly 150 degrees the boar turns normally, above it the turn would take long enough to look wrong, so it hops. A rebuild that picks a different threshold changes how a boar behaves when the player circles it.

## `net_Spawn`

**Contract** — Places the creature in the world. Installs the head-tracking hook on the head bone and zeroes the tracking state. Returns failure if the base spawn failed.

```text
FUNCTION net_Spawn(spawn_record) -> bool
  IF NOT base.net_Spawn(spawn_record)   RETURN false
  IF the creature has no ragdoll yet
    install_bone_hook(head_bone, BoneCallback, self)
  current_head_delta = 0
  target_head_delta  = 0
  head_turn_speed    = half a turn per second
  look_at_enemy      = false
  RETURN true
```

**Notes** — The hook is installed only when no ragdoll exists. Once the creature has a physics shell, the shell owns every bone transform and drives its own hooks; adding this one would either be overwritten or would fight the simulation. A rebuild whose animation and physics pose paths are separate still needs the same rule stated as: *the head-tracking offset applies to the animated pose only, never to a simulated one.*

## `BoneCallback`

**Contract** — Called by the animation layer once per frame, after the head bone's animated transform is computed and before it is used to skin or to derive world positions from. Post-multiplies a yaw of `-current_head_delta` onto that transform. Does nothing when head tracking is off. Must be cheap: it runs per creature per frame inside the pose evaluation.

```text
FUNCTION BoneCallback(bone)
  boar = bone.owner
  IF NOT boar.look_at_enemy   RETURN
  bone.transform = bone.transform COMPOSED_WITH yaw_rotation(-boar.current_head_delta)
```

**Notes** — The rotation is applied in the bone's own frame (post-multiplied), so the head turns about its own axis rather than swinging around the body's. The negated angle is a convention of the model's head axis, not a decision.

## `UpdateCL`

**Contract** — One frame of head tracking: slews `current_head_delta` toward `target_head_delta` at `head_turn_speed`, using the frame's own elapsed time so the rate is frame-rate independent, and taking the short way round the circle.

```text
FUNCTION UpdateCL()
  base.UpdateCL()
  current_head_delta = angular_lerp(current_head_delta, target_head_delta,
                                    head_turn_speed, frame_delta_seconds)
```

**Notes** — Nothing in this file writes `target_head_delta` or `look_at_enemy`; the behaviour states do. This function's only job is to make the head *lag* the target rather than snap to it.

## `CheckSpecParams`

**Contract** — The hook by which an animation's authored "special parameter" flags reach the creature. The boar implements none; the body is empty.

**Notes** — The retired implementation is preserved in the source as comments and is worth one line here because it documents an alternative: the jump-turn used to be driven from this hook, computing the heading to the enemy, choosing the left or right spin clip by which side the enemy was on, nudging the target heading a twentieth of a half-turn past the enemy so the spin overshoots rather than undershoots, and forcing the angular speed to two and a half times the angle divided by the clip's length so the spin finishes with the animation. That was replaced by the shared jump-turn ability configured in `reinit`. A second retired branch played a dedicated charging-attack clip; the creature keeps the ability flag but no longer has the clip.

## `~Boar`

**Contract** — Releases the state manager the constructor created. The constructor's other job — telling the "controllable" mixin which object it decorates — has no teardown.
