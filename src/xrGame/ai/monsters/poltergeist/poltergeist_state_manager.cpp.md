# src/xrGame/ai/monsters/poltergeist/poltergeist_state_manager.cpp

> The poltergeist's brain, and a study in a selector that was cut back: seven states are registered, the selector chooses between exactly two of them, and the elaborate attack it was written for survives only as a disabled routine.

**Needs** — [`poltergeist_state_manager.h`](poltergeist_state_manager.h.md) · [`poltergeist.h`](poltergeist.h.md) · [`poltergeist_state_rest.h`](poltergeist_state_rest.h.md) · [`poltergeist_state_attack_hidden.h`](poltergeist_state_attack_hidden.h.md) · [`../states/monster_state_eat.h`](../states/monster_state_eat.h.md) · [`../states/monster_state_panic.h`](../states/monster_state_panic.h.md) · [`../states/monster_state_hitted.h`](../states/monster_state_hitted.h.md) · [`../states/monster_state_hear_int_sound.h`](../states/monster_state_hear_int_sound.h.md) · [`../states/monster_state_hear_danger_sound.h`](../states/monster_state_hear_danger_sound.h.md)
**Used by** — [`poltergeist_state_manager.h`](poltergeist_state_manager.h.md)
**Tier floor** — T3: a two-branch selector

## Purpose

The root of the poltergeist's state tree. It registers a full creature's worth of states —
resting, eating, hidden attack, panic, reacting to a hit, and the two sound reactions — and
then selects among them with a rule that can only ever produce two of the seven.

The live rule is one line: **if there is an enemy and the detection level has passed the
circling threshold, attack while hidden; otherwise rest.** Everything else is unreachable.

That is worth stating as the design rather than as a defect, because it is what the creature
*is*: a poltergeist has no wounded reaction, no panic, and no interest in sound, and it cannot
be distracted from an alerted player. The five unreachable states are registered anyway, which
costs an allocation each and keeps the creature's script-visible state vocabulary complete.

## `execute` — the selector

**Contract** — chooses a state, runs it, and records it as the previous one. Called every tick
through the root's update guards.

```text
FUNCTION execute()
  IF enemy_manager.enemy exists AND creature.detected_enemy()
    state = attack_while_hidden
  ELSE
    state = rest

  select(state)
  current_state.run()
  previous_substate = current_substate
```

**Invariants** — selection is unconditional in both directions: unlike every other creature in
the chapter, the poltergeist's selector does **not** use the start-or-continue rule. A
poltergeist whose detection level drops below the threshold mid-attack returns to rest that
same tick, with no hysteresis at all. The detection level's own decay is the only thing
smoothing it.

**Notes** — the gate is `detected_enemy`, the *circling* threshold, not the ability's firing
threshold. The two are separate numbers and the circling one is lower, so the creature starts
orbiting before it starts attacking. The abilities check their own, higher threshold
themselves.

## The cut selector

The original selector survives in full as disabled code beside the live one, and it is worth
recording because it describes a creature the data files still carry parameters for:

- a visible poltergeist with an enemy chose panic against a strong threat and a **plain
  ground attack** against a weak one; only a hidden one used the hidden attack;
- being hit while visible drove the hit reaction;
- a dangerous sound drove the danger reaction while visible and the *interesting*-sound
  reaction while hidden — a hidden creature investigated rather than fled;
- otherwise it ate if it had a corpse, and resting **disabled hiding** once within five units
  of the corpse so the creature materialised to feed, re-enabling hiding on leaving the eating
  state.

The eating branch is the only one that explains an otherwise puzzling piece of live machinery:
the hiding lock in [`poltergeist.cpp`](poltergeist.cpp.md) exists for it, and nothing now uses
it.

## `polter_attack` — the cut scare routine

**Contract** — as shipped, does nothing. Its three cooldown timestamps are reset by `reinit`
and never advanced.

The disabled body describes three interleaved attacks on independent cooldowns, each cooldown
drawn between a minimum and one of two maxima depending on whether the creature's health had
fallen below half — an "aggressive" band and a normal one:

- a **flame** attack, gated on the enemy having been seen within the last five seconds;
- a **telekinesis** attack;
- a **scare**, choosing at random between the physical shove and the strange-sound effect
  documented in [`poltergeist_ability.cpp`](poltergeist_ability.cpp.md).

In the shipped creature, the flame and telekinesis attacks moved into the abilities themselves
where they run on their own clocks, and the scare pair was lost entirely — which is why both
scare implementations are complete and unreachable.

A rebuild targeting fidelity leaves all of this out and keeps the two-branch selector. A
rebuild targeting the intended creature has, in this routine, the only surviving statement of
what the intent was.
