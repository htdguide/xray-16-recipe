# src/xrGame/ai/monsters/burer/burer_state_attack_gravi_inline.h

> The wind-up is the weapon's range indicator: the further away the enemy, the longer the burer holds the charge before letting the wave go.

**Needs** — [`burer_state_attack_gravi.h`](burer_state_attack_gravi.h.md) · [`burer.h`](burer.h.md) · [`control_animation_base.h`](../control_animation_base.h.md) · [`control_direction_base.h`](../control_direction_base.h.md)
**Used by** — [`burer_state_attack_gravi.h`](burer_state_attack_gravi.h.md)
**Tier floor** — T2: a timed phase machine; the wave it launches is advanced in [`burer.cpp`](burer.cpp.md)

## Purpose

The burer's signature attack, as the *behaviour* half — charging, timing and release. The wave that results is a separate mechanism, advanced per frame by the creature itself; this state only launches it.

Its most interesting decision is that the charge duration is not a constant: it scales with how far outside the attack's minimum range the enemy is, so a close enemy is hit almost immediately and a distant one gets a visible, dodgeable wind-up.

## State

```text
RECORD BurerAttackGraviState
  phase                   : ENUM { started, charging, fire, wait_clip_end, completed }
  time_charge_started     : int
  next_gravi_allowed_tick : int    # stamped on entry, not on exit
  clip_end_tick           : int
```

Read from the creature: `gravi.min_dist`, `gravi.max_dist`, `gravi.time_to_hold`, `gravi.cooldown`, and the flag saying whether the model has one gravity clip or three. See [`burer.cpp`](burer.cpp.md).

## `initialize`

**Contract** — Resets the phase machine, stamps the cooldown forward from *now*, clears the scripted forced-attack override, and blocks the script layer from taking the creature while the attack runs.

**Notes** — Stamping the cooldown at entry rather than at exit means the pause is measured from when the attack *started*, so a long charge against a distant enemy eats into its own cooldown. That is why a burer harassing a distant player can chain waves more often than the cooldown number alone suggests.

## `execute` — the phase machine

**Contract** — One tick. Always faces the enemy first. Plays clips by *overriding* the animation selection rather than by asking for an action, because the charge has no corresponding abstract action.

```text
FUNCTION execute()
  face_enemy()
  triple = the model has three gravity clips

  IF phase == started
     force clip: gravity, variant 0
     IF time_charge_started not yet set
        clip_end_tick       = now() + length_of(gravity clip variant 0)
        time_charge_started = now()
        start the wind-up effect on the victim
     IF triple AND now() <= clip_end_tick     RETURN    # let the intro clip finish
     phase = charging

  ELSE IF phase == charging
     force clip: gravity, variant (triple ? 1 : 0)
     distance = |enemy - self|
     hold = clamp(|distance - min_dist| / min_dist, 0, 1) * time_to_hold
     IF time_charge_started + hold >= now()   RETURN
     phase = fire
     IF triple
        clear the forced clip; force gravity variant 2
        clip_end_tick = now() + length_of(gravity clip variant 2)

  ELSE IF phase == fire
     force clip: gravity, variant (triple ? 2 : 0)
     launch the wave from half a unit above own origin
                       toward half a unit above the enemy's origin
     stop the wind-up effect
     play the gravity attack sound
     phase = wait_clip_end

  ELSE IF phase == wait_clip_end
     IF now() > clip_end_tick   phase = completed
```

**Notes** — The hold formula is the whole tuning of the attack. It measures how far the enemy is *outside the minimum distance*, in units of that minimum distance, clamps that ratio to one, and multiplies by the authored hold time. So: at exactly the minimum distance the hold is zero and the wave fires instantly; at twice the minimum distance or beyond, the hold is the full authored time. A rebuild that makes the hold proportional to absolute distance instead gets a creature whose openings scale with the level's size rather than with its own tuning.

Both launch points are raised half a unit above the origins. The wave travels at roughly waist height because it is a floor-level shockwave, and launching from the origin would put it in the ground.

The single-clip and three-clip paths differ in more than which variant plays: the three-clip path waits for the intro clip to finish before it starts charging, and re-times the ending on the outro clip. With one clip, phases advance purely on the hold timer. Both must work, because which one applies depends on the model the data ships.

## `check_start_conditions`

**Contract** — Whether a gravity attack may begin.

```text
FUNCTION check_start_conditions() -> bool
  IF forced_gravi_attack_override             RETURN true     # scripts bypass everything
  IF now() < next_gravi_allowed_tick          RETURN false
  IF distance_to_enemy < min_dist             RETURN false
  IF distance_to_enemy > max_dist             RETURN false
  IF NOT can_see_enemy_right_now()            RETURN false
  IF NOT facing(enemy, within 45 degrees)     RETURN false
  RETURN true
```

**Notes** — The forced override returns before the cooldown is even read, which is deliberate: a scripted set-piece must be able to make a burer fire on cue regardless of what it has been doing. The override is one-shot — `initialize` clears it.

## `check_completion`

**Contract** — True once the phase machine reaches `completed`.

## `finalize` / `critical_finalize`

**Contract** — Both restore the creature to the script layer's reach, and the aborted path additionally stops the wind-up effect that would otherwise be left playing on the victim. The two differ, which is unusual in this chapter and correct here: a normal completion has already stopped the effect in the `fire` phase.
