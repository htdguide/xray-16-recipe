# src/xrGame/ai/monsters/controller/controller_state_control_hit_inline.h

> The controller's mind-control strike: start a three-part animation, break it at a fixed moment
> to deliver the hit, then wait for the animation to finish.

**Needs** — [`controller_state_control_hit.h`](controller_state_control_hit.h.md) · [`controller.h`](controller.h.md) · [`../anim_triple.h`](../anim_triple.h.md) · [`../control_manager_custom.h`](../control_manager_custom.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`controller_state_control_hit.h`](controller_state_control_hit.h.md)
**Tier floor** — T3: a phase machine over an animation clock

## Purpose

An ability wrapped in a state. The controller's attack is not a projectile and not a melee blow:
it is an animation with a moment inside it at which the victim is affected. This state exists to
own that timing — it starts the animation, waits a fixed interval, tells the animation to move on
from its looping middle part, and applies the effect in the same tick.

**Currently unreachable in play**: no state manager registers it, so the shipped controller
delivers its mind-control hit through another path.

## State

```text
RECORD ControlHitState
  phase                 : ENUM { prepare, continue, fire, wait_for_animation_end, completed }
  time_control_started  : int
```

## `CStateControlAttack`

**Contract** — one phase transition per tick, plus unconditional per-tick work: request the
standing idle action, face the enemy with a turn budget, and play the aggression sound at the
section's attack-sound delay. Finishes when the phase machine reaches its end. Starts only when
the enemy is currently visible **and** further away than the good-strike distance — this is a
ranged act and refuses at close quarters.

**Invariants** — the strike is applied at most once per activation, because the fire phase is
entered exactly once and immediately advances. The visibility test is repeated at the moment of
the strike, so an enemy that breaks line of sight during the wind-up is not hit even though the
animation plays out.

```text
FUNCTION execute()
  SWITCH phase
    CASE prepare
      begin_three_part_animation(section's control animation set)
      play_control_start_sound()            # heard inside the victim's head, not in the world
      time_control_started = now()
      phase = continue

    CASE continue
      IF time_control_started + PREPARE_TIME < now()
        phase = fire

    CASE fire
      advance_three_part_animation_past_its_loop()
      IF sees_enemy_now()
        apply_control_hit()
      phase = wait_for_animation_end

    CASE wait_for_animation_end
      IF NOT three_part_animation_active()
        phase = completed
      # falls through to completed in the same tick

    CASE completed
      # nothing

  request_action(stand_idle)
  face(enemy, turn_budget = 1200)
  sound.play(aggressive, delay = section.attack_sound_delay)
```

**Notes** — the wind-up interval of 2.9 seconds is authored in code and must match the length of
the animation's first part in the shipped model data; it is a *duplicate* of a fact that lives in
the animation, not a tunable. A rebuild that derives the moment from the animation's own length
is more robust and will not reproduce the original exactly if the two disagree.

The minimum-distance refusal (8 world units) is the opposite of the usual melee gate: this
ability is *worse* up close, because the wind-up leaves the creature stationary and exposed. The
predicate is a refusal, not a preference — nothing re-tries at a better distance.

The phase machine deliberately does not fall out of the wait phase on its own clock; it waits for
the animation system to report the animation gone. That couples the state's lifetime to data, not
to a timer, which is what lets modders swap the animation set without retuning the state.

The fall-through from the waiting phase into the completed phase is a C++ switch without a break,
and here it is harmless — both do nothing further in the tick. A rebuild should write the two
phases as distinct cases and not inherit the fall-through as a feature.
