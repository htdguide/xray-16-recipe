# src/xrGame/ai/monsters/poltergeist/poltergeist.cpp

> An invisible flying creature whose whole behaviour is gated on a *detection level* that the player's own movement near it builds up: it drifts unseen at a wandering height, and only once the player has stirred it enough does it circle and attack.

**Needs** — [`poltergeist.h`](poltergeist.h.md) · [`poltergeist_state_manager.h`](poltergeist_state_manager.h.md) · [`poltergeist_movement.h`](poltergeist_movement.h.md) · [`../monster_velocity_space.h`](../monster_velocity_space.h.md) · [`../basemonster/base_monster.h`](../basemonster/base_monster.h.md) · [`../energy_holder.h`](../energy_holder.h.md) · [`../telekinesis.h`](../telekinesis.h.md) · [Seam: Rigid-body dynamics](../../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Graphics device](../../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`poltergeist.h`](poltergeist.h.md)
**Tier floor** — T2: creates and destroys a physics body at runtime and hands a transform back to the rigid-body layer

## Purpose

Every other creature in chapter 24 is a body that walks. This one is a *presence*. Three
mechanisms make it, and none of them is shared with any other species:

**Hidden mode.** For almost all of its life the poltergeist is invisible and has **no physics
body at all** — the body is destroyed on entering hidden mode and re-created on leaving it.
While hidden, the creature's authoritative location is a position it carries itself, and its
rendered position is that plus a drifting height. What the player sees is a particle effect.

**The detection level.** A scalar that rises with how fast the player moves near the
poltergeist and falls at a constant rate otherwise, and gates everything: the creature only
circles once it is above one threshold, and its ability only fires above a higher one. A player
who creeps past a poltergeist is never noticed; a player who runs past wakes it. This inverts
the usual perception model — the creature does not look for the player, the player *provokes*
the creature.

**Drifting height.** While hidden the creature picks a new target height at random intervals
and eases towards it, which is what gives the effect its unsettled, floating look. The height
is not driven by navigation and has no effect on where the creature can go.

## State

```text
RECORD Poltergeist                      # on top of the shared creature base
  hidden                : bool          # no physics body, not rendered
  hiding_locked         : bool          # script pinned the mode; enter/leave requests are ignored
  graph_position        : vector        # the creature's position ON the navigation graph while hidden
  height                : real          # current drift height above graph_position
  target_height         : real
  height_retarget_at    : int           # global clock at which a new target is picked

  ability               : PolterAbility # exactly one, chosen at load

  detection_level       : real          # invariant: within [0, detection_max]
  last_detection_at     : int
  last_player_position  : vector
  detection_effect_slot : int           # 0 = no screen effect attached
  actor_ignore          : bool          # script switch: be inert towards the player

  # authored in the creature's configuration section
  invisible_gait        : (linear, angular)  # "Velocity_Invisible_Linear" / "_Angular"
  height_change_rate    : real   # "Height_Change_Velocity",  default 0.5
  height_retarget_min   : int    # "Height_Change_Min_Time",  default 3000 ms
  height_retarget_max   : int    # "Height_Change_Max_Time",  default 10000 ms
  height_min            : real   # "Height_Min",              default 0.4
  height_max            : real   # "Height_Max",              default 2.0
  circle_threshold      : real   # "detection_fly_around_level",  default 5
  circle_distance       : real   # "detection_fly_around_distance", default 15
  circle_turn_period    : real   # "detection_fly_around_change_direction_time", default 7 s
  ability_kind          : text   # "type" — "flamer" selects flame, anything else selects telekinesis
  detection_effect_name : text   # "detection_pp_effector_name"
  detection_near_gain   : real   # "detection_near_range_factor", default 2
  detection_far_gain    : real   # "detection_far_range_factor",  default 1
  detection_speed_power : real   # "detection_speed_factor",      default 1
  detection_decay       : real   # "detection_loose_speed",       default 5 per second
  detection_range       : real   # "detection_far_range",         default 20
  detection_threshold   : real   # "detection_success_level",     default 4
  detection_max         : real   # "detection_max_level",         default 100
```

Invariants: `hidden` true implies there is no physics body and the rendered position is
`graph_position` lifted by `height`. `detection_effect_slot` non-zero exactly while a screen
effect is attached to the player, and it must be released before the creature is destroyed.

## `Load`

**Contract** — reads the creature's section: the invisible gait (registered with the movement
layer as an extra velocity), the full animation set and its action bindings, the height and
circling numbers, the detection numbers, and — the one branching decision — **which ability to
construct**.

```text
IF section.type = "flamer" THEN ability = new PolterFlame  ELSE ability = new PolterTele
ability.load(section)
```

**Notes** — the ability is chosen once at load and never changes. Two different creatures in
the shipped data share this class and differ only in that word and in the parameters each
ability reads, which is the chapter's stated pattern — *differ by data, plus one distinctive
ability* — taken about as far as it goes.

The animation set is unusual in two ways. Several distinct actions map to the same clip
(`stand_idle_` serves idle, sitting, lying, sleeping, resting and dragging, and also the death
animation with a zero-length hold) because a floating creature has no posture. And the damaged
run reuses the damaged *walk* clip at run velocity, so a hurt poltergeist plays a walk cycle
while moving fast — a data shortcut, not a bug in the code.

Two configuration keys for a "fake psi aura" are read in commented-out form and are dead.

## `update_detection` — the provocation model

**Contract** — called every frame. Raises the detection level from the player's speed and
proximity, decays it, and attaches or detaches the screen effect that tells the player
something is stirring. Also publishes the level to the renderer as the player's *visibility to
this creature*.

```text
FUNCTION update_detection()
  IF the creature is dead, or there is no living player
    detach the screen effect
    RETURN

  player_pos = player.position
  distance   = straight_line(player_pos, creature.position)
  elapsed    = seconds since last_detection_at ; last_detection_at = now

  IF NOT actor_ignore AND 0 < elapsed < 2 AND distance < detection_range
    nearness   = distance / detection_range                      # 0 at contact, 1 at the edge
    range_gain = nearness * detection_far_gain + (1 - nearness) * detection_near_gain
    raw_speed  = distance travelled by the player since the last call / elapsed
    speed_term = (1 + raw_speed) ^ detection_speed_power - 1     # 0 when standing still
    immunity   = player.hit_immunity(telepathic)

    detection_level = detection_level + elapsed * 0.03 * immunity * range_gain * speed_term

  detection_level = max(0, detection_level - elapsed * detection_decay)
  detection_level = min(detection_level, detection_max)
  IF elapsed non-zero THEN last_player_position = player_pos

  publish_player_visibility(creature.identity, effect_factor())

  IF detection_level > 0.01 AND an effect name was authored
    attach the screen effect if not already attached, reading its intensity from effect_factor
  ELSE
    detach it
```

**Invariants** — a player who does not move contributes nothing, because the speed term is
exactly zero at zero speed. That is the mechanic: **stillness is invisibility**.

**Notes** — the elapsed-time window of two seconds discards a single long gap, which is what a
level load or a pause produces; without it the first frame after a resume would deliver a huge
apparent speed. The lower bound excludes a zero-length frame.

The range gain interpolates from the near gain at contact to the far gain at the edge of range,
so with the defaults (near 2, far 1) closeness doubles the rate.

The speed term raises one-plus-speed to an authored power and subtracts one, which keeps it
zero at rest for any exponent while letting a species be tuned to notice sprinting
disproportionately.

The constant `0.03` is the master gain of the whole system and is not authored, not named, and
not derived anywhere. It is the single number that decides how long a running player takes to
wake a poltergeist, and it is the most significant unexplained constant in this file.

Scaling by the player's telepathic *immunity* means better psi protection makes the player
**more** detectable, since the immunity is a multiplier that rises with protection in this
engine's convention. That reads backwards and no rationale is recoverable.

The creature publishes the normalised level as the player's visibility, which is what drives
the renderer's own reveal of the creature — so the player seeing the poltergeist and the
poltergeist noticing the player are literally the same number.

## `detected_enemy`

**Contract** — whether the detection level has passed the *circling* threshold, which is a
different and lower number than the ability's firing threshold. So a poltergeist wakes and
begins to circle before it can attack.

## Hidden mode

**Contract** — `hide` and `show`, both idempotent, both refused while hiding is locked.

```text
FUNCTION hide()
  IF already hidden THEN RETURN
  hidden = true
  stop rendering
  graph_position = current position          # remember where the body was
  destroy the physics body
  ability.on_hide()

FUNCTION show()
  IF not hidden THEN RETURN
  hidden = false
  start rendering
  play the materialise animation as a one-shot sequence
  position = graph_position                   # come back where the graph says we are
  re-create the physics body at that position
  ability.on_show()
```

**Invariants** — the graph position is the authoritative location across the transition in both
directions. The physics body exists exactly while not hidden.

**Notes** — the creature **spawns hidden**: both the spawn path and the reset path enter hidden
mode explicitly, destroy the body, and start the ability's hidden particle effect. A rebuild
that spawns it visible gets a poltergeist standing on the floor.

Hiding and showing are driven by the energy mixin's automatic activation, so the creature
oscillates between hidden and visible on the energy budget's own schedule — but the **hiding
lock** overrides that, and is what an eating or scripted poltergeist uses to stay materialised.

## `update_height`

**Contract** — while hidden only, re-randomises the drift target whenever the retarget time has
passed, picking a new height within the authored band and a new interval within the authored
time band. The height itself eases towards the target every frame at the authored rate.

**Notes** — the retarget instant is computed as the current clock plus a random interval, and
both the height and the next interval are drawn fresh each time, so the motion has no period.
That irregularity is the intended look.

The initial height of 0.3 — set in three places on spawn and reset — sits below the authored
minimum of 0.4, so the creature always drifts *upward* from its resting height on first wake.
Nothing derives 0.3.

## Per-frame and per-tick updates

**`update_frame`** — detection first, then the creature base's own frame work, then ease the
height towards its target, then the ability's frame work. Finally, if the player can see the
creature and is within 85 world units, the creature asks the renderer to treat it as a small
always-visible object. That distance is unexplained; its effect is to keep a distant
poltergeist from being culled and popping in.

**`update_schedule`** — releases the screen effect if the creature or the player has died, then
the base's scheduled work, the telekinesis mixin's, the energy mixin's, the height retarget,
and the ability's scheduled work. Note it does **not** return early when the work condition
fails — everything still runs on a dead poltergeist except the effect.

## `Die`

**Contract** — a telekinetic poltergeist that dies while hidden is made visible *at its graph
position*, transform pushed into the physics layer if a ragdoll exists, so the corpse falls
from where the creature actually was rather than from wherever the body was last left. Then
telekinesis is deactivated, the energy budget disabled, and the ability told.

**Notes** — the correction is guarded on the creature being a *telekinetic* one, not on being
hidden, which means a flame poltergeist dying while hidden leaves its corpse at the stale body
position. Whether that asymmetry was deliberate is not recoverable.

## `Hit`

**Contract** — the ability is told first (it spawns a damage particle effect), then, **if the
player was the attacker, the detection level is slammed to its maximum**. Shooting a
poltergeist wakes it instantly and completely, bypassing the whole gradual provocation model.
Then the creature base handles the damage.

## `create_movement_manager`

**Contract** — installs the poltergeist's own path-follower, described in
[`poltergeist_movement.cpp`](poltergeist_movement.cpp.md), in place of the shared one, and
registers it as both the path controller and the base controller.

## `force_final_animation`

**Contract** — while hidden, pins the animation to the floating clip. This is the hook the
animation layer calls when nothing else has chosen a clip, and without it a hidden poltergeist
would fall back to a ground pose it has no body for.

## Debug overrides

Six of the detection numbers are read through a per-creature debug override hook rather than
directly, so they can be re-tuned on a live creature. In a release build each is the identity.
Incidental, except that it documents which six numbers the authors expected to iterate on:
the two range gains, the speed exponent, the decay, the range and the threshold.
