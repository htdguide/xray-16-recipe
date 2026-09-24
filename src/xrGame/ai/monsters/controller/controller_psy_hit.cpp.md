# src/xrGame/ai/monsters/controller/controller_psy_hit.cpp

> The set-piece psi attack: four clips during which the player's weapons are blocked, the camera is hauled toward the creature, the player is thrown backwards, and psi damage lands — all of it scaled by the player's psi resistance.

**Needs** — [`controller_psy_hit.h`](controller_psy_hit.h.md) · [`controller.h`](controller.h.md) · [`../control_animation_base.h`](../control_animation_base.h.md) · [`../control_direction_base.h`](../control_direction_base.h.md) · [`../control_movement_base.h`](../control_movement_base.h.md) · [`../basemonster/base_monster.h`](../basemonster/base_monster.h.md) · [`../../../Actor.h`](../../../Actor.h.md) · [`../../../ActorCondition.h`](../../../ActorCondition.h.md) · [`../../../ActorEffector.h`](../../../ActorEffector.h.md) · [Seam: Rigid-body dynamics](../../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Static collision database](../../../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — reached through its declarations in [`controller_psy_hit.h`](controller_psy_hit.h.md); callers name that, not this file.
**Tier floor** — T2: drives a camera effector, a physics impulse and a network event

## Purpose

The controller's signature attack, and the only ability in this slice that takes the player's
body as well as the creature's. It is a custom control element like any other — it seizes the
four body resources and plays clips — but between the clips it reaches out and does four
things to the player: blocks every weapon, installs a camera effector that drags the view
toward the creature and narrows the field of view, throws the player backwards with an
impulse, and lands psi damage.

Three of those four are **scaled by the player's psi resistance**, which is the attack's
whole balance: a player with good psi protection sees a shallower camera pull and a narrower
field-of-view change. The damage and the impulse are not scaled.

The attack targets **the actor specifically**, not "the creature's enemy". Every condition
and every effect names the player. It is a set piece, not a general attack.

## State

```text
RECORD PsyHit
  stage           : list<Motion> of exactly four       # psy_attack_0 .. psy_attack_3
  current_index   : int                                # 0..3
  sound_state     : one of { prepare, start, pull, hit, none }
  min_tube_dist   : real                               # authored, shared with the creature
  blocked         : bool                               # the player's weapons are currently blocked
  time_last_tube  : int (ms)                           # cooldown stamp
```

**Invariants** — the weapon block is raised in stage one and lowered on release, never on the
stage that raised it. So every abort path — the creature dying, the player dying, the enemy
being lost — still gives the player their weapons back. That is the same discipline the
critical wound uses and for the same reason.

## The four stages

The clips are resolved by literal name and each transition is driven by an animation-end
event.

| Stage | Clip | What happens when it *starts* |
|---|---|---|
| 0 | `psy_attack_0` | the wind-up; nothing beyond the clip |
| 1 | `psy_attack_1` | the whole set piece opens: see `death_glide_start` |
| 2 | `psy_attack_2` | the sound state advances to the hit sounds |
| 3 | `psy_attack_3` | the damage lands: see `death_glide_end` |

After stage three the element deactivates itself.

## `check_start_conditions`

**Contract** — five tests, all naming the actor: the ability is not running; no other ability
holds the creature's body; no psi-hit camera effector is already installed on the actor; the
creature can see the actor right now; the cooldown has expired; and the creature is **at
least** the authored minimum distance from the actor.

**Notes** — testing for the existing camera effector is how two controllers are stopped from
performing the set piece on the same player at once. The minimum distance, here as in the
creature's own condition, is a floor rather than a ceiling: the drag needs room to be
visible.

## `activate`

**Contract** — seize all four body resources, subscribe to animation end, stop the path and
the movement, aim the creature's heading at the actor at three radians per second, start the
first clip, clear the weapon-block flag, and begin the preparation sound.

## `death_glide_start` — the set piece opens

**Contract** — runs as stage one begins. First re-checks the conditions and aborts if
anything changed. Then:

```text
FUNCTION death_glide_start()
  IF NOT check_conditions_final() THEN deactivate; RETURN

  hide the heads-up display
  install the actor hold on this creature, and freeze its camera turn

  REQUIRE no psi-hit camera effector is already installed

  src    <- the actor's camera position
  target <- my position, raised 1.2 units
  dir    <- normalize(target - src); dist <- |target - src|

  resistance <- the actor's psi hit immunity
  # the camera is pulled from src toward me, stopping 4.8 units short at zero resistance
  # and barely moving at full resistance
  target <- src + dir * (0.01 + resistance * (dist - 4.8))

  fov_from <- the actor's field of view
  fov_to   <- fov_from - (fov_from - 10) * resistance

  install a camera effector interpolating src -> target and fov_from -> fov_to
          over the duration of this stage's clip

  draw the creature's psi particle effect

  # throw the actor backwards and upwards
  away <- normalize(src - target), pitched up by a sixth of a turn
  apply an impulse of (actor mass * 530) along `away`

  start the attack sounds
  send the actor a "block all weapons" event; record that we blocked
  re-aim the creature's heading at the actor at three radians per second
```

**Notes** — the resistance scaling reads backwards on first sight and is worth stating
carefully. At **zero** resistance the camera travels almost the whole way and stops 4.8 units
short of the creature, and the field of view collapses toward ten degrees — the player is
hauled in and tunnel-visioned. At **full** resistance the camera moves one hundredth of a
unit and the field of view does not change — the player barely notices. So *higher
resistance means less effect*, and the interpolation is written as a lerp on the resistance
rather than on its complement.

The 4.8-unit standoff, the hundredth-of-a-unit floor, the ten-degree field of view, the
impulse coefficient of 530 and the sixth-of-a-turn upward pitch are all constants in this
file with no recorded derivation. The impulse is scaled by the actor's mass, so it is a
velocity change rather than a force, and 530 is the only number here that is obviously in
physical units.

Freezing the camera turn immediately after installing the hold is deliberate: the hold's own
easing would fight the camera effector, so the hold is used purely to suppress input.

The weapon block is sent as a **network event to the actor** rather than as a direct call.
That is a multiplayer-shaped mechanism in a single-player set piece; a rebuild may call
directly, but the block and the unblock must remain paired.

## `death_glide_end` — the hit

**Contract** — runs as stage three begins: draw the particle effect again, play the two hit
sounds hard left and hard right in two-dimensional space, apply the creature's authored tube
damage as psi damage to the actor, stamp the cooldown, and stop.

**Notes** — playing the two channels at fixed opposite positions in two-dimensional sound
space is a deliberate stereo effect: the hit arrives in both ears at once rather than from a
direction. Every sound in this attack is played that way, which is what makes it read as
happening inside the player's head.

## `check_conditions_final`

**Contract** — the re-check at the top of stage one: the creature is alive, an actor exists,
the actor is one of the creature's enemies, the actor is alive, the creature is at least the
minimum distance minus two units away horizontally, and the creature can see the actor.

**Notes** — the distance is measured *horizontally* here and in three dimensions in the start
condition, and the threshold is relaxed by two units. Both differences are deliberate slack:
the player may have moved during the wind-up clip, and failing the set piece after it has
visibly begun looks worse than continuing.

A commented-out condition would have required the actor to be the creature's *current* enemy
rather than merely an enemy. It was loosened.

## `stop` / `on_death` / `deactivate`

**Contract** — `stop` restores the heads-up display and releases the actor hold, and removes
the camera effector. `on_death` runs it and deactivates, so a creature killed mid-attack does
not leave the player blind and disarmed. `deactivate` releases the body, unsubscribes, lowers
the weapon block if it was raised, and silences the sounds.

**Notes** — the division of labour between `stop` and `deactivate` is the reason this attack
is safe to interrupt. `stop` undoes what was done to the *player's view*; `deactivate` undoes
what was done to the player's *input* and to the creature's body. The two are reached
independently and both are idempotent.

## `set_sound_state`

**Contract** — a small state machine over the creature's five attack sounds: preparation
starts one; the start state stops it and begins the drag and whoosh together; the hit state
stops those two; the none state stops all three. Every sound plays at the actor, in
two-dimensional space.

**Notes** — the hit state's own two sounds are commented out here and played from
`death_glide_end` instead, which is why the sound state machine has a stage that only stops
things.

## `see_enemy`

**Contract** — can the creature see the actor right now, asked through the creature's own
enemy memory.

**Notes** — a much stricter version is present and commented out: it traced four rays between
the two heads and the two centres and required **all four** to reach the actor. It was
abandoned for the single visibility query. That is a meaningful loosening — the shipped
attack can begin through a gap the player is only partly visible through.

## `tube_ready` / `load` / `reinit` / `update_frame`

**Contract** — `tube_ready` is the cooldown check, reading the creature's authored minimum
delay and falling back to five seconds if the element's creature is not a controller.
`load` reads the minimum distance from the creature's section under the same key the creature
reads it with. `reinit` resolves the four clips, each required, and clears the index, the
cooldown stamp and the sound state. `update_frame` is empty — everything is driven by
animation-end events.

**Notes** — the five-second fallback delay can never be reached, since only a controller owns
this element. It is defensive and harmless.
