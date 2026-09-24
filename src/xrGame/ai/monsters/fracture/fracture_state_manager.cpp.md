# src/xrGame/ai/monsters/fracture/fracture_state_manager.cpp

> The fracture's brain: the solitary selector with two states missing — it cannot be controlled,
> and it does not distinguish an interesting sound from a dangerous one.

**Needs** — [`fracture_state_manager.h`](fracture_state_manager.h.md) · [`fracture.h`](fracture.h.md) · [`../monster_state_manager.h`](../monster_state_manager.h.md) · [`../states/monster_state_rest.h`](../states/monster_state_rest.h.md) · [`../states/monster_state_attack.h`](../states/monster_state_attack.h.md) · [`../states/monster_state_panic.h`](../states/monster_state_panic.h.md) · [`../states/monster_state_eat.h`](../states/monster_state_eat.h.md) · [`../states/monster_state_hitted.h`](../states/monster_state_hitted.h.md) · [`../states/monster_state_hear_danger_sound.h`](../states/monster_state_hear_danger_sound.h.md)
**Used by** — [`fracture_state_manager.h`](fracture_state_manager.h.md)
**Tier floor** — T3: a priority selector, once per creature update

## Purpose

The smallest brain in the chapter: six global states, all generic, and the standard ordering with
two omissions. Comparing it against
[`../flesh/flesh_state_manager.cpp`](../flesh/flesh_state_manager.cpp.md) — the reference solitary
brain — is the fastest way to see what a creature can leave out.

## State

The manager owns no data. Six registered global states, all generic: rest, attack, eat, heard a
dangerous sound, panic, was hit.

Absent by comparison with the reference brain: **under another creature's control** (a fracture
cannot be taken over, matching its class, which is not a controllable entity), **heard a call for
help** (it is not a pack creature and has nobody to call), and **heard an interesting sound** as a
separate state.

## `execute`

**Contract** — choose one global state, switch into it, execute it, record it as the previous
state.

```text
FUNCTION execute()
  IF an enemy is known
    state = (danger_rating(enemy) == strong) ? panic : attack
  ELSE IF hit_memory_is_fresh()  THEN state = was_hit
  ELSE IF heard an interesting sound OR heard a dangerous sound
    state = hear_danger                      # both sounds route to the same response
  ELSE IF a corpse is available and we are hungry THEN state = eat
  ELSE                                       state = rest

  switch_to(state)
  active_state.execute()
  previous = state
```

**Notes** — two decisions, both subtractive.

**Every sound is a dangerous sound.** The base creature's perception still distinguishes the two
kinds — the flags are separate and both are read — but the fracture routes them to one state. So a
fracture never investigates: it hides from a dropped bolt the same way it hides from a gunshot.
That is a characterisation, not an oversight, and it is one line.

**Fear is unconditional.** Unlike the flesh, the fracture does not suppress panic when it has been
hit; a strong enemy makes it flee no matter how much damage it has taken. Combined with the sound
collapse, the creature is timid in every branch that admits timidity.

The selector reads the danger rating from the creature's enemy memory, which weighs the enemy
against the creature's own section — so *which* enemies count as strong is authored in data even
though the response to them is in code.
