# src/xrGame/ai/monsters/flesh/flesh_state_manager.cpp

> The flesh's brain: the plain solitary-creature priority selector, with one twist — a flesh
> facing a strong enemy flees, unless it has already been hurt, in which case it fights.

**Needs** — [`flesh_state_manager.h`](flesh_state_manager.h.md) · [`flesh.h`](flesh.h.md) · [`../monster_state_manager.h`](../monster_state_manager.h.md) · [`../states/monster_state_rest.h`](../states/monster_state_rest.h.md) · [`../states/monster_state_attack.h`](../states/monster_state_attack.h.md) · [`../states/monster_state_panic.h`](../states/monster_state_panic.h.md) · [`../states/monster_state_eat.h`](../states/monster_state_eat.h.md) · [`../states/monster_state_hitted.h`](../states/monster_state_hitted.h.md) · [`../states/monster_state_controlled.h`](../states/monster_state_controlled.h.md) · [`../states/monster_state_help_sound.h`](../states/monster_state_help_sound.h.md) · [`../states/monster_state_hear_int_sound.h`](../states/monster_state_hear_int_sound.h.md) · [`../states/monster_state_hear_danger_sound.h`](../states/monster_state_hear_danger_sound.h.md)
**Used by** — [`flesh_state_manager.h`](flesh_state_manager.h.md)
**Tier floor** — T3: a priority selector, once per creature update

## Purpose

This is the **reference solitary brain** for the chapter. Nine global states, all generic, and a
selector that is the plain version of the ordering every creature varies: being controlled beats
everything, then combat, then damage, then sounds, then appetite, then idling. Read this one
first; every other creature's brain in this chapter is this one plus a named difference.

## State

The manager owns no data. Its nine registered global states are all the generic implementations:
rest, panic, attack, eat, heard an interesting sound, heard a dangerous sound, heard a call for
help, was hit, and under another creature's control.

## `execute`

**Contract** — choose one global state, switch into it, execute it, and record it as the previous
state. Exactly one state is chosen on every update; there is no path that leaves the brain idle.

```text
FUNCTION execute()
  IF under_another_creature_control()
    state = controlled
  ELSE IF an enemy is known
    # the twist: fear only survives until the creature is actually hurt
    state = (danger_rating(enemy) == strong AND NOT hit_memory_is_fresh())
            ? panic : attack
  ELSE IF hit_memory_is_fresh()        THEN state = was_hit
  ELSE IF a call for help is pending   THEN state = hear_help
  ELSE IF heard an interesting sound   THEN state = hear_interesting
  ELSE IF heard a dangerous sound      THEN state = hear_danger
  ELSE IF a corpse is available and we are hungry THEN state = eat
  ELSE                                 state = rest

  switch_to(state)
  active_state.execute()
  previous = state
```

**Notes** — the one decision worth stating is the conjunction in the combat branch. Every creature
that can panic asks the same question — is this enemy rated strong? — and the flesh adds *and I
have not been hit*. The effect is that a flesh bolts from an armed player it has spotted, but a
flesh that takes a shot turns and charges. That reads, correctly, as a cornered animal, and it
prevents the frustrating case of a creature that flees forever while being shot.

The danger rating itself is not computed here; it comes from the creature's enemy memory, which
weighs the enemy's kind and armament against the creature's own section. So "strong" is authored
in data even though the *response* to it is in code.

Note that the flesh's combat branch consults the hit memory and the following branch consumes it:
a hit therefore both suppresses panic and, when there is no enemy, selects the was-hit state. The
ordering is what keeps those two uses from conflicting.

Compare [`../fracture/fracture_state_manager.cpp`](../fracture/fracture_state_manager.cpp.md),
which is the same selector with two states missing and no such conjunction, and
[`../dog/dog_state_manager.cpp`](../dog/dog_state_manager.cpp.md), which replaces the combat
branch with a pack-wide latch.
