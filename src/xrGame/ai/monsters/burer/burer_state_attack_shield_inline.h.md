# src/xrGame/ai/monsters/burer/burer_state_attack_shield_inline.h

> Put the shield up, stand and face the player behind it, and drop it when the authored time runs out — or early, if the player is reloading.

**Needs** — [`burer_state_attack_shield.h`](burer_state_attack_shield.h.md) · [`burer.h`](burer.h.md) · [`control_animation_base.h`](../control_animation_base.h.md)
**Used by** — [`burer_state_attack_shield.h`](burer_state_attack_shield.h.md)
**Tier floor** — T3: a timed state

## Purpose

The behaviour half of the burer's shield. Selected by the attack tree only when the creature has just been hurt — see [`burer_state_attack_inline.h`](burer_state_attack_inline.h.md) — which is what makes the shield read as a flinch rather than a stance.

## State

```text
RECORD BurerShieldState
  shield_started_tick   : int
  next_particle_tick    : int
  raise_clip_length_sec : real   # measured at entry, never read
  raised                : bool
```

Read from the creature: `shield_time`, `shield_cooldown`, `shield_keep_particle`, `shield_keep_particle_period`.

## `initialize`

**Contract** — Stamps the start time, clears the particle cooldown and the raised flag, blocks the script layer from taking the creature, and measures the raise clip's length.

**Notes** — The measured clip length is never used. The commented-out condition beside the raise in `execute` shows what it was for: the shield was to come up only *after* the raise animation had played, so the visual and the invulnerability would line up. As shipped, the shield comes up on the state's first tick and the raise clip plays over an already-protected creature. That is one frame of generosity to the creature and a visible mismatch a rebuild may choose to fix — but fixing it makes the shield strictly worse, and the shipped balance assumes the generous version.

## `execute`

**Contract** — One tick: raise the shield on the first tick, re-spawn the keep-alive effect on its authored period, face the enemy, stand, and force the raise or the sustain clip according to whether the shield is up.

```text
FUNCTION execute()
  IF NOT raised
     raised = true
     creature.activate_shield()
  IF raised AND keep_particle is configured AND now() > next_particle_tick
     spawn keep_particle one unit above own origin, attached to self
     next_particle_tick = now() + keep_particle_period
  face_enemy()
  action = stand_idle
  force clip: raised ? shield_sustain : shield_raise
```

**Notes** — The keep-alive effect is re-spawned on a period rather than started once, because the effect is authored as a short burst; the period is how a continuous shimmer is built out of it. A creature whose section names no keep-alive effect simply has an invisible shield, and several shipped configurations do.

## `check_start_conditions`

**Contract** — Refuses until the previous shield's duration *and* its cooldown have both elapsed since the last raise, and refuses if the enemy cannot currently see the creature.

```text
FUNCTION check_start_conditions() -> bool
  IF now() < shield_started_tick + shield_time + shield_cooldown   RETURN false
  IF NOT enemy_can_see_me_now()                                    RETURN false
  RETURN true
```

**Notes** — Both halves matter. Measuring the gap from the last *raise* rather than the last drop means the cooldown is a true period, not a rest interval. And the visibility test is inverted from the usual direction — it asks whether the *enemy* can see the creature, not the reverse — because a shield raised against someone who cannot see you is wasted. That check is why a burer never shields while behind cover.

## `check_completion`

**Contract** — Down when the authored duration has elapsed, or as soon as the creature's early-drop test says the player is reloading. See `CanDeactivateShieldEarly` in [`burer.cpp`](burer.cpp.md).

## `finalize` / `critical_finalize`

**Contract** — Both lower the shield and restore the script layer's reach. Lowering on both paths is the important part: a shield left up because the state was torn down would make the creature permanently invulnerable.
