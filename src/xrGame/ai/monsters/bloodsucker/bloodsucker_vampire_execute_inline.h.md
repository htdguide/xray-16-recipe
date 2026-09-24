# src/xrGame/ai/monsters/bloodsucker/bloodsucker_vampire_execute_inline.h

> The feed: take the player's camera and inventory away, hold for a fixed time, land the drain, give control back — and give it back on *every* exit path.

**Needs** — [`bloodsucker_vampire_execute.h`](bloodsucker_vampire_execute.h.md) · [`bloodsucker_vampire_effector.h`](bloodsucker_vampire_effector.h.md) · [`state.h`](../state.h.md) · [`anim_triple.h`](../anim_triple.h.md) · [`control_direction_base.h`](../control_direction_base.h.md) · [`control_path_builder_base.h`](../control_path_builder_base.h.md) · [Seam: Rigid-body dynamics](../../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`bloodsucker_vampire_effector.h`](bloodsucker_vampire_effector.h.md) · [`bloodsucker_vampire_execute.h`](bloodsucker_vampire_execute.h.md)
**Tier floor** — T2: behaviour, timing and camera arithmetic; the only low-level reach is asking the skeleton where a named bone is

## Purpose

This is the one place in the creature code where a creature seizes the player. For the duration of the feed the player cannot look, cannot move the camera, cannot use the inventory and has no heads-up display; the creature drives the camera to look at its own head bone. Because a seizure that is not released leaves the game unplayable, the release path is duplicated across every way this state can end — normal completion, the parent tree re-selecting, and the creature dying mid-feed all run the same cleanup.

The feed is performed by a *triple animation*: a start clip, a looping middle, and an end clip, with a scripted break point in the middle that is where the damage actually lands. The state does not play clips itself; it activates the triple and watches it.

## State

```text
RECORD VampireExecuteState
  phase             : ENUM { prepare, continue, fire, wait_triple_end, completed }
  time_hold_started : int       # set when the triple animation is activated
  effects_started   : bool      # the screen effects are started once, and only
                                # after the player's camera has finished turning
```

Authored numbers, fixed in code rather than in the creature's configuration section:

```text
hold_duration        = 4000 ms   # how long the drain runs before it lands
screen_effect_length = 6 s       # lifetime of both the camera and post-process effects
head_turn_cone       = 20 degrees  # see `execute`
```

## `initialize`

**Contract** — Seizes the player and starts the feed. Installs the creature as the player's control owner, points the player's view at the creature's head bone, makes the creature fully visible (it cannot feed while cloaked), hides the heads-up display, blocks the whole inventory and disables inventory input. Resets the hit counter that gates the *next* feed and draws a fresh random requirement for it. Sets the phase to `prepare` and marks the screen effects as not yet started. Allocates nothing that outlives the state.

**Invariants** — After this runs the player is under this creature's control; the matching release in `cleanup` must run before the state is left by any path.

```text
FUNCTION initialize()
  actor_control.install(self)
  look_at_own_head_bone()
  phase = prepare
  time_hold_started = 0
  set_visibility(full)          # a feeding bloodsucker is never partly cloaked
  hits_taken_before_vampire = 0
  hits_required_before_next_vampire = random integer in [-1, 1]
  hide_hud()
  block_all_inventory_slots()
  disable_inventory_input()
  effects_started = false
```

**Notes** — The random requirement drawn here is the *next* feed's precondition, not this one's: the creature must absorb a small, unpredictable number of hits between feeds so a player cannot be drained back to back. The draw is `-1 + random(0..2)`, i.e. one of −1, 0 or 1; a negative value means the next feed needs no hits at all, which is what makes the gate feel irregular rather than metered.

## `execute`

**Contract** — Runs one tick of the feed. Starts the screen effects the first tick after the player's forced camera turn has settled — starting them during the turn would fight the turn. Keeps the player's view pinned to the head bone, advances the phase machine, and keeps the creature facing and closing on the player. Does not block.

```text
FUNCTION execute()
  IF NOT actor_control.is_turning() AND NOT effects_started
    start_vampire_screen_effects()      # camera drag + post-process pulse
    effects_started = true

  look_at_own_head_bone()

  IF phase == prepare
    activate_triple_animation(vampire_clip_set)
    time_hold_started = now()
    play_sound(vampire_grasp)
    phase = continue
  ELSE IF phase == continue
    do_continue()
  ELSE IF phase == fire
    break_triple_animation_at_point()   # the damage instant of the clip
    play_sound(vampire_hit)
    satisfy_vampire()                   # heals the creature, clears its want
    phase = wait_triple_end
  ELSE IF phase == wait_triple_end
    IF NOT triple_animation_active()
      phase = completed
  # phase == completed: nothing

  face(enemy)
  IF angle_between(own_facing, direction_to_enemy) < head_turn_cone
     AND distance_to_enemy > configured_vampire_distance
    # aligned but drifted out of reach: run the gap down without breaking the hold
    action = run
    acceleration = aggressive, braking off
    path_target = enemy's navigation vertex and its position
    path_rebuild_interval = 100 ms
    path_use_covers = false
    path_stop_short_by = configured_vampire_distance
  ELSE
    action = stand_idle
```

```text
FUNCTION do_continue()
  IF NOT melee_reach_to(enemy)          # the player broke away
    deactivate_triple_animation()
    phase = completed
    RETURN
  play_sound(vampire_sucking)           # the sound layer rate-limits repeats
  IF time_hold_started + hold_duration < now()
    phase = fire
```

**Notes** — The creature only chases while it is *already* aimed at the player; misaligned, it stands and turns instead. That ordering is what stops the feed from degenerating into a circling run. The chase asks the pathfinder to stop short by exactly the configured feed distance, so arriving means arriving in reach.

## `check_start_conditions`

**Contract** — Answers whether the feed may begin. Consulted by the parent tree once the approach has closed. Reads only; no side effects.

```text
FUNCTION check_start_conditions() -> bool
  IF NOT enough_hits_absorbed_since_last_feed()          RETURN false
  IF NOT walkable_straight_line_to(enemy)                RETURN false
  IF NOT melee_reach_to(enemy)                           RETURN false
  IF NOT facing(enemy, within quarter turn)              RETURN false
  IF NOT wants_to_feed()                                 RETURN false
  IF enemy is not the player                             RETURN false
  IF player is already controlled by some creature       RETURN false
  IF player's input is already held by something else    RETURN false
  RETURN true
```

**Notes** — The straight-line test asks the navigation mesh whether the run from the creature's vertex toward the player stays on walkable ground; a feed started across a gap would drag the player through geometry. Two separate checks guard the player's input — one for "some creature owns the player" and one for "some other system owns the player's input" — because scripted sequences take input without going through the creature path.

## `check_completion`

**Contract** — True once the phase machine has reached `completed`. Nothing else ends this state from the inside; the parent tree ends it from the outside.

## `finalize` / `critical_finalize`

**Contract** — Both run the same cleanup, which is the point: whether the feed ended on its own or was torn down mid-hold, the player must come back. Re-enable inventory input, stop the triple animation if it is still running, release the player if this creature still owns them, restore the heads-up display and unblock the inventory slots.

**Invariants** — After either, the player owns their own camera, input and inventory, and no triple animation is left active on this creature.

**Notes** — The release is guarded by "if this creature still owns the player" rather than done unconditionally, because a second creature may have taken the player over in the meantime and releasing then would cut that feed short.
