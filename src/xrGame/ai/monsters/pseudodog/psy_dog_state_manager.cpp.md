# src/xrGame/ai/monsters/pseudodog/psy_dog_state_manager.cpp

> The psy dog's brain is the pseudodog's brain with one question asked first: am I short of illusions, and is the player the one hunting me? If so, hide and make more.

**Needs** — [`psy_dog_state_manager.h`](psy_dog_state_manager.h.md) · [`psy_dog.h`](psy_dog.h.md) · [`psy_dog_state_psy_attack.h`](psy_dog_state_psy_attack.h.md) · [`pseudodog_state_manager.h`](pseudodog_state_manager.h.md) · [`Actor.h`](../../../Actor.h.md)
**Used by** — [`psy_dog_state_manager.h`](psy_dog_state_manager.h.md)
**Tier floor** — T3: one test in front of an inherited selector

## Purpose

This is the whole of what separates the psy dog's behaviour from the pseudodog's. One state is registered and one condition is tested; when the condition is false the creature *is* a pseudodog, running the inherited selector unchanged.

Writing the difference this way rather than as a rewritten selector is the pattern the chapter uses for most of its creatures, and it is why so few of them need a substantial brain file. See [`README.md`](../README.md).

## State

The brain owns no data of its own. It adds one registered global state — the psychic attack — to those the pseudodog brain registers.

## `execute` — the selector

**Contract** — if the creature's enemy is the player *and* the creature is below its configured minimum number of live illusions, select the psychic attack, run it, and record it. Otherwise defer entirely to the pseudodog brain's selector.

```text
FUNCTION execute()
  enemy = my current enemy
  IF enemy is the player AND my live illusion count < section.minimum_illusions
    select(psy_attack) ; psy_attack.execute() ; previous = psy_attack
  ELSE
    pseudodog_brain.execute()
```

**Invariants** — exactly one branch runs and each leaves an active state, so the brain is never left without one.

**Notes** — the enemy must be *the player specifically*, not any enemy. Illusions of the creature only exist to deceive a viewer, and the machinery that judges whether the deception is working reads the player's perception. Against another creature the psy dog fights as an ordinary pseudodog.

The threshold is "fewer illusions alive than my configured minimum", so the creature does not hide once and produce a fixed number of illusions — it returns to hiding every time the player has destroyed enough of them, which is what makes the fight a war of attrition against copies rather than a single trick.

The counting is of *live* illusions, so the condition is driven by how successful the player has been, not by elapsed time. A player who ignores the illusions never sees the creature hide again.
