# src/xrGame/ai/monsters/pseudodog/pseudodog_state_manager.cpp

> The pseudodog's brain: a flat priority selector over seven states, and the chapter's reference example of what a creature selector looks like when nothing unusual is going on.

**Needs** — [`pseudodog_state_manager.h`](pseudodog_state_manager.h.md) · [`pseudodog.h`](pseudodog.h.md) · [`../states/monster_state_rest.h`](../states/monster_state_rest.h.md) · [`../states/monster_state_attack.h`](../states/monster_state_attack.h.md) · [`../states/monster_state_panic.h`](../states/monster_state_panic.h.md) · [`../states/monster_state_eat.h`](../states/monster_state_eat.h.md) · [`../states/monster_state_hitted.h`](../states/monster_state_hitted.h.md) · [`../states/monster_state_hear_int_sound.h`](../states/monster_state_hear_int_sound.h.md) · [`../states/monster_state_hear_danger_sound.h`](../states/monster_state_hear_danger_sound.h.md)
**Used by** — [`pseudodog_state_manager.h`](pseudodog_state_manager.h.md)
**Tier floor** — T3: a five-branch selector

## Purpose

Seven states, all of them the shared implementations with no pseudodog-specific behaviour, and
a selector that is a plain priority chain. This is the baseline against which every other
creature in the chapter is a variation, and it is worth reading before any of them.

## `execute` — the selector

**Contract** — chooses a state, runs it, records it as previous. Called every tick through the
root's update guards.

```text
FUNCTION execute()
  enemy = enemy_manager.enemy

  IF enemy exists
    state = CASE enemy_manager.danger_type OF
              strong : panic          # the odds are bad: run
              weak   : attack         # the odds are good: fight
  ELSE IF hit_memory.has_hits()
    state = react_to_hit             # hurt by something unseen
  ELSE IF heard_interesting_sound
    state = investigate_sound
  ELSE IF heard_dangerous_sound
    state = react_to_danger_sound
  ELSE IF can_eat()
    state = eat
  ELSE
    state = rest

  select(state)
  current_state.run()
  previous_substate = current_substate
```

**Invariants** — the priority is absolute and, as with the poltergeist's selector, there is
**no start-or-continue test at any level**: the chain is re-evaluated from the top every tick
and the chosen state is selected unconditionally. Persistence, where a creature has it, lives
*inside* the chosen state's own sub-selector, not here. So a dog that acquires an enemy
abandons its meal in the same tick, and a dog whose enemy dies drops out of attack immediately.

**Notes** — the ordering is the whole design and each step is a claim about what a dog cares
about:

- **A known enemy outranks everything**, including being hurt. A dog being shot by something it
  can see fights or flees; it never plays the wounded reaction.
- **Fight or flee is decided by the odds, not by health or morale.** The danger type is the
  coarse verdict computed in
  [`../monster_enemy_manager.cpp`](../monster_enemy_manager.cpp.md) from the number of allies
  and enemies nearby, which is what makes a lone dog flee and a pack of four attack the same
  target. This is the single most characteristic line in the file.
- **Interesting sound outranks dangerous sound.** Curiosity beats caution, which is inverted
  relative to every other creature in the chapter that has both, and is why dogs come to
  investigate gunfire rather than avoiding it.
- **Eating outranks resting** but is below every reaction, so a dog eats only when nothing at
  all is happening.

The danger-type branch has no default: a `none` verdict, which the enemy manager produces when
the target was just acquired and while a target is *forced*, leaves the chosen state
unassigned. The selection then falls through with an invalid identifier. Nothing guards it; in
practice the verdict is set on the same tick the enemy is. A rebuild should default it to the
attack state.

Two constants sit unused in this file — a minimum angry duration and a maximum growling
duration — which belong to the unimplemented anger model described in
[`pseudodog.cpp`](pseudodog.cpp.md).
