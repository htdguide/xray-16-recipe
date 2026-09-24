# src/xrGame/ai/monsters/poltergeist — the poltergeist

Part of chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../../../SYSTEM-REQUIREMENTS.md#7-build-order).
Shared machinery: the [chapter opener](../../README.md) and the [creature layer](../README.md).

An invisible flying creature that **never touches its target**. It has no melee, no leap and
no body the player can shoot while it is hidden. Everything it does is at a distance and
through an ability.

## The three ideas

**A detection level the player builds himself.** The creature's behaviour is gated on a
running measure that the player's own movement near it raises. Below the threshold the
poltergeist drifts unseen at a wandering height doing nothing; above it, it circles and
attacks. So a player who moves carefully can pass one, and a player who blunders about wakes
it. That is a perception model inverted — the creature's state depends on what the *player*
has done, not on what the creature has sensed.

**Two interchangeable abilities behind one creature.** A poltergeist carries either
telekinesis or flame, chosen by its configuration section, sharing a common base that
supplies the particles, the idle sound and two ambient scares. The two attacks are completely
different mechanisms with the same trigger, which is why the ability is a separate object
rather than a branch in the brain.

- *Telekinesis*: a four-phase cycle that lifts nearby physics objects one at a time, holds
  them, then hurls them one at a time at the player's head — and only objects with a clear
  line to him, so the attack cannot be blocked by geometry it would have hit anyway.
- *Flame*: columns of fire raised at navigable points **near the player**, each running a
  three-phase lifecycle and burning anything the player-aimed ray passes through.

**Its attack is a movement pattern and nothing else.** The one attack state orbits the target
at an authored radius, reversing direction on a timer, shrinking the orbit when the level
will not accommodate it and growing it back when it will. The damage comes entirely from the
ability, which runs independently.

## An unusual piece of plumbing

While hidden the creature has no physical body, so the shared path follower has nothing to
move. The poltergeist therefore carries **its own path follower** that advances a graph
position instead. A rebuilder must not fold this back into the shared one; it exists
precisely because the shared one assumes a body.

## What could not be recovered

- **The brain was cut back and the cuts are visible.** Seven states are registered and the
  selector chooses between exactly two of them. The elaborate attack selection it was written
  for survives as a disabled routine. Nothing says why.
- The resting state is the shared one with its eating and sleeping branches removed, leaving
  a three-step priority over where to be. Whether the removal was deliberate or a stub is not
  recoverable.

## Twins

| Twin | Role |
|---|---|
| [`poltergeist.cpp`](poltergeist.cpp.md) | An invisible flying creature whose whole behaviour is gated on a *detection level* that the player's own movement near it builds up: it drifts unseen at a wandering height, and only once the player has stirred it enough does it circle and attack. |
| [`poltergeist.h`](poltergeist.h.md) | Declares the poltergeist — an invisible flying creature that never touches its target — together with the two interchangeable abilities it attacks through and the shared base those abilities extend. |
| [`poltergeist_ability.cpp`](poltergeist_ability.cpp.md) | The shared half of a poltergeist's ability — the particle effects and idle sound every poltergeist has whichever ability it carries — plus the two ambient scares the creature performs directly. |
| [`poltergeist_flame_thrower.cpp`](poltergeist_flame_thrower.cpp.md) | The flame poltergeist's attack: columns of fire raised at navigable points *near the player*, each running a three-phase lifecycle and burning anything the player-aimed ray passes through. |
| [`poltergeist_movement.cpp`](poltergeist_movement.cpp.md) | Advances a hidden poltergeist along its route by moving a position it carries itself, because while hidden it has no physical body for the shared follower to move. |
| [`poltergeist_movement.h`](poltergeist_movement.h.md) | Declares the poltergeist's own path follower, which while hidden moves a graph position rather than a body. |
| [`poltergeist_script.cpp`](poltergeist_script.cpp.md) | Registers the poltergeist's type name with the script layer, and nothing else. |
| [`poltergeist_state_attack_hidden.h`](poltergeist_state_attack_hidden.h.md) | Declares the poltergeist's only attack state: circle the target at a distance while the abilities do the damage. |
| [`poltergeist_state_attack_hidden_inline.h`](poltergeist_state_attack_hidden_inline.h.md) | The poltergeist's attack, which is a movement pattern and nothing else: orbit the target at an authored radius, reversing direction on a timer, shrinking the orbit when the level will not accommodate it and growing it back when it will. |
| [`poltergeist_state_manager.cpp`](poltergeist_state_manager.cpp.md) | The poltergeist's brain, and a study in a selector that was cut back: seven states are registered, the selector chooses between exactly two of them, and the elaborate attack it was written for survives only as a disabled routine. |
| [`poltergeist_state_manager.h`](poltergeist_state_manager.h.md) | Declares the poltergeist's brain: seven registered states and a selector with only two live outcomes. |
| [`poltergeist_state_rest.h`](poltergeist_state_rest.h.md) | The poltergeist's idle state: the shared resting behaviour with its eating and sleeping branches removed, leaving a strict three-step priority over where to be. |
| [`poltergeist_telekinesis.cpp`](poltergeist_telekinesis.cpp.md) | The telekinetic poltergeist's attack: a four-phase cycle that lifts nearby physics objects one at a time, holds them, then hurls them one at a time at the player's head — but only objects that have a clear line to the player. |
