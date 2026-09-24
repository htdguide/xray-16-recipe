# src/xrGame/ai/monsters/pseudodog — the pseudodog and the psy dog

Part of chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../../../SYSTEM-REQUIREMENTS.md#7-build-order).
Shared machinery: the [chapter opener](../../README.md) and the [creature layer](../README.md).

Two creatures share this directory, and the pair is the best single illustration in the
chapter of the claim that **a creature is usually its base plus one ability**.

The **pseudodog** is the reference creature: a leaping pack predator with a flat priority
selector over seven states and nothing unusual anywhere. A rebuilder who wants to see what a
plain creature selector looks like should read its brain first and every other creature's
second.

The **psy dog** extends it. Its entire behavioural difference is *one* registered state and
*one* condition in front of the inherited selector: if the player is hunting me and I have
fewer illusions alive than my configured minimum, hide. Everything else about it is a
pseudodog.

## The illusion mechanic

The psy dog fights by proxy. It keeps a population of illusory copies of itself alive; each
copy spawns unseen, materialises in the player's face with a leap and a jolt, and dissolves
at the first hit from its target. Meanwhile the real animal stays out of sight.

Three decisions hold that together.

**The hiding trigger counts *live* copies, not elapsed time.** The dog returns to hiding
every time the player has destroyed enough of them, so the fight is a war of attrition
against copies. A player who ignores the illusions never sees the dog hide again.

**The illusions are real creatures.** They have brains, perception and enemies of their own —
not particles with a script. That is why they can be flanked, why they behave differently in
a corridor than in the open, and why the dread rule below can ask what they have noticed.

**The dread is the link.** A screen treatment tells the player that the illusions around him
are not accidents, on a rule that reads *both* directions of awareness — has the player seen
a copy recently, and has a copy noticed the player recently — gated on the *real* animal
being within thirty units. Since the proximity test is on the real one, a player surrounded
by copies feels nothing once the animal has withdrawn, which is the tell that lets an
attentive player work out where it is.

## What could not be recovered

- Both windows in the dread rule (two seconds after the player sees a copy, ten seconds after
  a copy notices the player), the thirty-unit radius and the five-second fade are fixed in
  code. Only the *look* of the effect is configured.
- The dread's fade-out restarts its clock rather than carrying the strength already reached,
  so an effect cut short mid-fade briefly gets stronger before clearing. Whether that was
  noticed is not recoverable.
- **Both halves of the pseudodog's psi effector are empty files.** The howl exists; whatever
  screen effect was to accompany it does not.

## Twins

| Twin | Role |
|---|---|
| [`pseudodog.cpp`](pseudodog.cpp.md) | A leaping pack predator: the chapter's baseline creature with a full posture set, a corpse-dragging gait, a psi howl, and a leap that can rotate a quarter turn in mid-air. |
| [`pseudodog.h`](pseudodog.h.md) | Declares the pseudodog: a leaping pack predator that can drag corpses and perform a psi howl, and the base the psi dog extends. |
| [`pseudodog_psi_effector.cpp`](pseudodog_psi_effector.cpp.md) | Empty. |
| [`pseudodog_psi_effector.h`](pseudodog_psi_effector.h.md) | Empty. |
| [`pseudodog_script.cpp`](pseudodog_script.cpp.md) | Registers the three dog types with the script layer. |
| [`pseudodog_state_manager.cpp`](pseudodog_state_manager.cpp.md) | The pseudodog's brain: a flat priority selector over seven states, and the chapter's reference example of what a creature selector looks like when nothing unusual is going on. |
| [`pseudodog_state_manager.h`](pseudodog_state_manager.h.md) | Declares the pseudodog's brain: seven states and a plain priority selector. |
| [`psy_dog.cpp`](psy_dog.cpp.md) | A dog that fights by proxy: it keeps a population of illusory copies of itself alive, hides while too few are up, and each copy spawns unseen, materialises in the player's face with a leap and a jolt, and dissolves at the first hit from its target. |
| [`psy_dog.h`](psy_dog.h.md) | Declares the psi dog and its phantom: a pseudodog that conjures illusory copies of itself and hides behind them, and the copy that materialises in the player's face and vanishes when struck. |
| [`psy_dog_aura.cpp`](psy_dog_aura.cpp.md) | The psy dog's presence as a feeling rather than an attack: while its illusions and the player are aware of each other and the real creature is close, the player's screen curdles — and it eases off a few seconds after that stops being true. |
| [`psy_dog_aura.h`](psy_dog_aura.h.md) | Declares the psy dog's ambient dread: a fading screen treatment on the player, and the rule that decides when it should be on. |
| [`psy_dog_state_manager.cpp`](psy_dog_state_manager.cpp.md) | The psy dog's brain is the pseudodog's brain with one question asked first: am I short of illusions, and is the player the one hunting me? If so, hide and make more. |
| [`psy_dog_state_manager.h`](psy_dog_state_manager.h.md) | Declares the psy dog's brain: the pseudodog brain with one extra state in front of it. |
| [`psy_dog_state_psy_attack.h`](psy_dog_state_psy_attack.h.md) | Declares the psy dog's illusion attack: a one-node composite whose only job is to get the real animal out of sight. |
| [`psy_dog_state_psy_attack_hide.h`](psy_dog_state_psy_attack_hide.h.md) | Declares the psi dog's one hiding move: sprint to a cover point chosen relative to the enemy, then stop. |
| [`psy_dog_state_psy_attack_hide_inline.h`](psy_dog_state_psy_attack_hide_inline.h.md) | The psi dog's retreat: pick a cover point that hides you from where the enemy is, run there at full aggression, and stop when you are standing on it. |
| [`psy_dog_state_psy_attack_inline.h`](psy_dog_state_psy_attack_inline.h.md) | The psi dog's "attack": a container with exactly one child, re-entered forever — the dog's contribution to the fight is to keep relocating, and the phantoms do the fighting. |
