# src/xrGame/ai/monsters/monster_morale.cpp

> A creature's willingness to fight, as one number that drifts at a rate chosen by an externally-set mode and is nudged by being hit and by landing a hit.

**Needs** — [`monster_morale.h`](monster_morale.h.md) · [`monster_morale_inline.h`](monster_morale_inline.h.md)
**Used by** — [`monster_morale.h`](monster_morale.h.md)
**Tier floor** — T3: a clamped scalar integrated against elapsed time

## Purpose

Morale is the state variable behind "this creature has had enough". It is deliberately not
health: a creature at full health can be broken by a bad fight, and a wounded one can rally.

The design has two layers that a rebuild must keep separate. The **value** moves continuously,
at a rate that depends on a **mode** — stable, taking heart, or despondent — and the mode is set
from outside, by the brain, not derived here. So this file never decides that a creature is
losing; it only integrates the consequence once something else has said so. On top of the drift
sit two discrete nudges, one for taking a hit and one for landing one.

## State

```text
RECORD Morale
  creature             : BaseMonster
  mode                 : enum { stable, taking_heart, despondent }
  value                : real                 # invariant: within [0, 1]

  # authored in the creature's configuration section; all six required
  hit_penalty          : real   # "Morale_Hit_Quant"
  success_bonus        : real   # "Morale_Attack_Success_Quant"
  rate_taking_heart    : real   # "Morale_Take_Heart_Speed",  units per second, upward
  rate_despondent      : real   # "Morale_Despondent_Speed",  units per second, downward
  rate_stable          : real   # "Morale_Stable_Speed",      units per second, upward
  despondent_threshold : real   # "Morale_Despondent_Threashold"
```

Invariants: `value` is clamped into the unit interval on every write, drift and nudge alike.
After `reinit` the mode is stable and the value is full — a creature always starts willing.

## `load`

**Contract** — reads the six numbers from the creature's configuration section. All required;
there are no defaults.

**Notes** — the threshold key is spelled `Morale_Despondent_Threashold` in the shipped data
files. The misspelling is frozen by that data and cannot be corrected in a rebuild that loads
the original game.

## `on_hit` / `on_attack_success`

**Contract** — subtract the hit penalty, or add the success bonus, clamping. Instant, unrelated
to the mode.

**Notes** — the two magnitudes are authored independently, which is what lets a species be
tuned as one that breaks easily but recovers on a kill, or the reverse.

## `update_schedule`

**Contract** — the continuous drift, called with the milliseconds elapsed since the previous
call — the creature's own update interval, which varies with distance and load, not a fixed
tick. Rate is chosen by mode; the despondent rate is applied negatively.

```text
FUNCTION update_schedule(elapsed_ms)
  rate = CASE mode OF
           stable       : rate_stable          # drifts UP
           taking_heart : rate_taking_heart    # drifts UP, faster
           despondent   : -rate_despondent     # drifts DOWN

  value = value + rate * elapsed_ms / 1000
  clamp value into [0, 1]
```

**Invariants** — stable is a *recovery* mode, not a hold: a creature left alone climbs back to
full morale at the stable rate. There is no mode in which morale is stationary, so a creature
is always either recovering or breaking.

**Notes** — the elapsed time is converted to seconds by an integer division in the original, so
an update interval under a thousand milliseconds contributes **nothing at all**. A creature
updated frequently — which is to say a creature near the player — therefore drifts more slowly
than a distant one, or not at all. This is very likely unintended, and it is also what the
shipped creatures were tuned against. Reproduce it, and flag it; a rebuild that uses real
division will find every creature's morale moving far faster than the authored numbers imply.
